---
layout: post
title: "Ceph FileStore"
description: ""
category: ceph
tags: [ceph]
---
{% include JB/setup %}

# Introduction

ceph后端存储引擎有多种实现(filestore, kstore, memstore, bluestore), bluestore将来会成为默认的后端存储，
但是需要一些时间，现在大部分部署都是使用filestore。filestore的代码还是比较好理解的，
执行流程可以参考网上的[这篇文章](http://blog.csdn.net/ywy463726588/article/details/42679869)。

在阅读代码的过程中，一些细节还是需要注意，比如不同PG的OP操作可以并行执行，同一PG内部OP请求必须串行执行，
各个限流组件怎么协调工作，journal回放的时候需要注意非幂等操作。

本文顺着代码流程，分析写流程中一些值得关注的细节，然后总结下throttle， 非幂等操作和tuning参数。

涉及到的相关线程:

* OSD::osd\_op\_tp -> 提交写请求到journal队列

* FileJournal::write thread -> 写journal

* JournalingObjectStore::finisher -> journal完成后的回调

* FileStore::ondisk_finisher -> journal落盘的回调，标志写成功，但数据不可读

* FileStore::op_tp -> apply到文件系统的page cache，不保证落盘

* WBThrottle::thread -> apply文件系统的限流

* FileStore::op_finisher -> apply文件系统完成的回调，标志数据可读

* FileStore::sync thread -> sync文件系统的内容到磁盘，将序列号通知journal，使得journal可以释放空间，重复利用

操作语义:

* submit: 提交到journal队列

* apply: 写文件系统page cache

* commit: 将文件系统page cache数据sync到磁盘

备注一下，这里的commit指sync线程将page cache的数据sync到`data disk`的意思，不是journal的disk。容易引起误解的地方是，osd\_op\_tp在提交事务的时候，
会有两个回调，一个是ondisk(客户端可以认为是on commit)，表示数据已经落盘，因为journal一般采用O\_DIRECT + O\_DSYNC方式，写journal成功就表示数据落盘，
可以调用ondisk回调，通知客户端写成功，所以journal能改善写的性能，将随机转化为顺序，并且多个写可以合并成一次journal的写。
另外一个是onreadable，表示数据可读，在FileStore::op\_tp线程池将数据写入page cache后，就可以读数据，可以调用onreadable回调。

重要数据结构:

```cpp
class JournalingObjectStore : public ObjectStore {
protected:

  class SubmitManager {
    Mutex lock;
    uint64_t op_seq; // journal提交的序列号，全局唯一
    uint64_t op_submitted;
	......
  } submit_manager;

  class ApplyManager {
    Mutex apply_lock;
    bool blocked;
    Cond blocked_cond;
    int open_ops;
    uint64_t max_applied_seq; // apply到文件系统page cache的序列号

    Mutex com_lock;
    map<version_t, vector<Context*> > commit_waiters;
    uint64_t committing_seq, committed_seq; // 将文件系统page cache的数据fsync到磁盘的序列号，用来通知journal释放空间
	......
  } apply_manager;

  ......
};
```

# Write

## OSD::osd\_op\_tp

OSD::osd\_op\_tp线程池在执行PG写操作的时候，是通过函数queue\_transactions提交请求的:

```cpp
int FileStore::queue_transactions(Sequencer *posr, list<Transaction*> &tls,
				  TrackedOpRef osd_op,
				  ThreadPool::TPHandle *handle)
{
  ......

  // posr，定义在PG类中，对于同一个PG，一定是一样的，这里的default_osr根本就不会用，Jewel新代码已经删除了
  // OpSequencer非常关键，同一个PG会使用同样的OpSequencer，保证PG操作串行化
  OpSequencer *osr;
  if (!posr)
    posr = &default_osr;
  if (posr->p) {
    osr = static_cast<OpSequencer *>(posr->p);
  } else {
    osr = new OpSequencer; // PG的第一次操作的时候，会创建一个OpSequencer，以后就会复用
    osr->parent = posr;
    posr->p = osr;
  }

  ......

  if (journal && journal->is_writeable() && !m_filestore_journal_trailing) {
    Op *o = build_op(tls, onreadable, onreadable_sync, osd_op);

    op_queue_reserve_throttle(o, handle); // filestore层对整个op的限流，释放的时候是在FileStore::_finish_op

    journal->throttle(); // 对journal的限流

    // 在journal层面为op生成唯一sequence, 因为journal是单线程写，所以写一定是串行的
    uint64_t op_num = submit_manager.op_submit_start();
    o->op = op_num;

    if (m_filestore_journal_parallel) {
		......
    } else if (m_filestore_journal_writeahead) { // ext4, xfs都需要wal
      
      osr->queue_journal(o->op); // journal层面的sequence记录在OpSequencer中的journal queue中, JournalingObjectStore::finisher线程中会deque_journal

      _op_journal_transactions(o->tls, o->op, // 提交OP，注意这里的callback以及sequence
			       new C_JournaledAhead(this, osr, o, ondisk),
			       osd_op);
    } else {
      assert(0);
    }
    submit_manager.op_submit_finish(op_num); // op提交到journal队列完成
    return 0;
  }

  ......
}

void JournalingObjectStore::_op_journal_transactions(
  list<ObjectStore::Transaction*>& tls, uint64_t op,
  Context *onjournal, TrackedOpRef osd_op)
{
  if (journal && journal->is_writeable()) {
	......
    journal->submit_entry(op, tbl, data_align, onjournal, osd_op); // 放入journal队列，等待write线程执行journal写请求
  } else if (onjournal) {
    apply_manager.add_waiter(op, onjournal);
  }
}

void FileJournal::submit_entry(uint64_t seq, bufferlist& e, int alignment,
			       Context *oncommit, TrackedOpRef osd_op)
{
  ......

  // 获取journal限流资源
  throttle_ops.take(1);
  throttle_bytes.take(e.length());

  {
	// 注意锁的顺序
    Mutex::Locker l1(writeq_lock);  // ** lock **
    Mutex::Locker l2(completions_lock);  // ** lock **

	// write线程执行完成后，会处理这里的completion
    completions.push_back(
      completion_item(
	seq, oncommit, ceph_clock_now(g_ceph_context), osd_op));

    if (writeq.empty())
      writeq_cond.Signal();

    writeq.push_back(write_item(seq, e, alignment, osd_op)); // 放入队列，等待write线程执行
  }
}
```

执行到这里，请求已经提交到journal的队列里面，OSD::osd\_op\_tp工作就结束了。

## FileJournal::write\_thread

写journal线程通过Filejournal::write\_thread完成，流程比较简单，执行完成后，就会调用:

```cpp
void FileJournal::queue_completions_thru(uint64_t seq)
{
  ......

  // journal的一次写可以同时写入多个op请求日志，所以这里是循环处理所有已经完成的op
  // 将回调全部放入finisher线程的队列
  while (!completions_empty()) {
    completion_item next = completion_peek_front();
    if (next.seq > seq) // sequence判断是否已经写入完成
      break;
    completion_pop_front();

	......
    if (next.finish) // 放入finisher队列，等待回调
      finisher->queue(next.finish); // finisher线程实际上是JournalingObjectStore中的finisher
	......
  }
  finisher_cond.Signal();
}
```

需要注意的是，在journal准备写和写完成处理completions的时候，调用队列的锁太频繁，可以优化。
master branch已经有类似patch: [pr6701](https://github.com/ceph/ceph/pull/6701)

## JournalingObjectStore::finisher

这里的回调就是C\_JournaledAhead，然后会执行下面这个函数，主要干两件事情：1）将op放入filestore队列排队 2）将ondisk回调放入FileStore::ondisk\_finisher：

```cpp
void FileStore::_journaled_ahead(OpSequencer *osr, Op *o, Context *ondisk)
{
  queue_op(osr, o); // 将op在filestore层面排队，准备写入文件系统

  list<Context*> to_queue;
  osr->dequeue_journal(&to_queue); // journal已经写成功，出队列

  // journal写好了，数据就真正落盘了，所以执行ondisk回调
  // 注意此时数据还未写入文件系统，所以不可读
  if (ondisk) {
    ondisk_finisher.queue(ondisk); // 放入ondisk_finisher的队列，等待回调
  }

  if (!to_queue.empty()) {
    ondisk_finisher.queue(to_queue);
  }
}
```

journal是单线程顺序执行的，且每条op请求都有唯一的sequence，使得queue\_op一定是按提交时候的顺序调用。
但是同一个PG可能连续提交了很多次op请求，这些请求会放入PG对应的OpSequencer中进行排队，然后同时将OpSequencer放入
op\_wq队列等待FileStore::op\_tp执行，所以如果PG连续提交请求，OpSequencer会在op\_wq中同时出现多次，
op\_tp中可能多个线程同时获取同一个OpSequencer准备执行写文件系统的操作:

```cpp
void FileStore::queue_op(OpSequencer *osr, Op *o)
{
  // queue_op按提交时候的顺序调用，必然导致属于同一个OpSequencer的OP按照提交顺序
  // 在OpSequencer内部排队, 保证了PG内部op的先后顺序
  osr->queue(o);
  op_wq.queue(osr);
}
```

## FileStore::op\_tp

PG对应的OpSequencer排队以后，说明PG有OP需要执行，这时候线程池就会对其处理，入口函数:

```cpp
void FileStore::_do_op(OpSequencer *osr, ThreadPool::TPHandle &handle)
{
  wbthrottle.throttle(); // filestore层面writeback的限流

  ......

  // op_tp线程池的多个线程可以并发对同一个OpSequencer执行请求
  // 锁保证同一个OpSequencer中(也即PG中）只能有一个OP在执行
  osr->apply_lock.Lock();

  Op *o = osr->peek_queue(); // 获取一个op

  apply_manager.op_apply_start(o->op);
  int r = _do_transactions(o->tls, o->op, &handle); // 执行写请求到文件系统
  apply_manager.op_apply_finish(o->op);
}
```

写执行完成后，线程还会执行一个finish函数:

```cpp
void FileStore::_finish_op(OpSequencer *osr)
{
  list<Context*> to_queue;
  Op *o = osr->dequeue(&to_queue); // 将op从OpSequencer出队列
  osr->apply_lock.Unlock();  // 释放锁，这时候其他线程就可以继续对此QpSequencer执行apply操作

  op_queue_release_throttle(o); // 释放filestore的throttle，见queue_transactions

  if (o->onreadable_sync) {
    o->onreadable_sync->complete(0);
  }
  if (o->onreadable) {
    op_finisher.queue(o->onreadable);
  }
  if (!to_queue.empty()) {
    op_finisher.queue(to_queue); // 放入op_finisher队列，等待执行apply回调，标志数据可读
  }
  delete o;
}
```

OpSequencer中apply\_lock保证PG内部OP的串行化，并不是保证内部队列q和jq的互斥，q和jq的互斥是另外一把锁qlock在保证。
所以在apply的过程中，OSD::osd\_op\_tp可以继续向jq中提交请求，更重要的是，JournalingObjectStore::finisher线程可以继续将
写journal完成的op在q中排队。

如前所述，同一个OpSequencer可能进入FileStore::op\_wq多次，然后被多个FileStore:op\_tp中的线程获取执行，然后q和jq共用一把锁，是否会影响性能？
其实也还好，虽然FileStore::osd\_tp是线程池，会有多个线程，但是这些线程在开始处理apply
的时候，会先获取apply\_lock，然后在执行完成的时候，从q出队列op的时候获取qlock，所以不会同时出现多个FileStore::osd\_tp的线程去
抢qlock这个锁，可以认为同一时刻q只增加了两个线程去抢qlock，即JournalingObjectStore::finisher 和 其中一个FileStore::osd\_tp线程。

## FileStore::sync_thread

sync线程实现比较简单，目的是获取一个序列号，保证此序列号之前的数据都已经apply过了，即数据已经在page cache中，
然后执行fsync，更新序列号，这样可以保证此序列号之前的数据已经存入disk中，以后不在需要，journal可以做trim释放空间。

需要注意在获取序列号的过程中，会导致FileStore::op\_tp block住，影响apply流程，对性能有损失，
可以适当调整参数filestore\_max\_sync\_interval。有一个潜在问题是，如果长时间不sync，可能会导致执行sync的时候，
整个目录数据过多，导致一次sync时间太长，也可能导致系统内存不足而OOM，这些需要结合kernel参数dirty\_ratio 和 dirty\_expire\_centisecs调优。

```cpp
void FileStore::sync_entry()
{
  lock.Lock();
  while (!stop) {
	......

    op_tp.pause(); // 暂停apply线程池的处理
    if (apply_manager.commit_start()) { // 如果有新的请求需要commit, 返回true

      uint64_t cp = apply_manager.get_committing_seq(); // 获取已经apply过的序列号

	  ......

      if (backend->can_checkpoint()) {
		  ......

      } else {
		apply_manager.commit_started(); // 设置block为false，主要是为journal replay服务
		op_tp.unpause(); // 恢复线程池

		int err = backend->syncfs(); // 这里会sync osd的整个current目录

		err = write_op_seq(op_fd, cp); // 记录下commit的序列号
	
		err = ::fsync(op_fd); // 保证更新序列号的操作落盘
      }
      
      apply_manager.commit_finish(); // 完成commit，通知journal
      wbthrottle.clear();

	  ......

    } else {
      op_tp.unpause();
    }
	......
  }
  stop = false;
  lock.Unlock();
}

bool JournalingObjectStore::ApplyManager::commit_start()
{
  bool ret = false;

  uint64_t _committing_seq = 0;
  {
    Mutex::Locker l(apply_lock);

    blocked = true; // 这个仅仅为journal replay起作用

    while (open_ops > 0) { // 等待其他inflight apply 完成
      blocked_cond.Wait(apply_lock);
    }

    {
      Mutex::Locker l(com_lock);
      if (max_applied_seq == committed_seq) {
		blocked = false;
		goto out;
      }

      _committing_seq = committing_seq = max_applied_seq; // 更新序列号
    }
  }
  ret = true;

 out:
  if (journal)
    journal->commit_start(_committing_seq);  // tell the journal too
  return ret;
}
```

这里比较晦涩的地方是，sync线程先pause住FileStore::op\_tp线程池，然后调用commit\_start(),
pause后说明线程池不会再有新的apply请求了，为什么还设置变量blocked为true？

首先，设置这个变量为true，目的是防止继续apply:

```cpp
uint64_t JournalingObjectStore::ApplyManager::op_apply_start(uint64_t op)
{
  Mutex::Locker l(apply_lock);
  while (blocked) { // 新的apply操作将会阻塞
    blocked_cond.Wait(apply_lock);
  }

  assert(!blocked);
  assert(op > committed_seq);
  open_ops++;
  return op;
}
```

其次，有一种特殊情况，即journal在做replay的时候，apply的操作不是在FileStore::op\_op线程池内完成，
而是在其他线程调用mount的时候，回放日志完成apply，所以pause op\_tp不起作用，停止不了apply操作。
如果回放日志太多或太久，导致sync线程开始工作，那么此时需要将回放日志的线程暂停一下，
以便获取序列号，这时候blocked就起作用了，可以阻塞调用mount的线程，等commit完成后唤醒继续replay。


```cpp
void JournalingObjectStore::ApplyManager::commit_started()
{
  blocked = false; // 设置回false
  blocked_cond.Signal(); // 唤醒
}
```

另外还需要注意，sync线程调用commit\_start()是有可能被阻塞的，需要等所有的inflight apply完成，
所以apply完成后会检查是否有blocked，这里和刚才的情况不一样，虽然都是阻塞在blocked变量上:

```cpp
void JournalingObjectStore::ApplyManager::op_apply_finish(uint64_t op)
{
  ......
  if (blocked) {
    blocked_cond.Signal(); // 唤醒sync线程
  }
  ......
}
```

# Throttle

FileStore实现中，提供了三个限流的地方:

* journal

* filestore apply

* filestore writeback

## journal

```cpp
int FileStore::queue_transactions(Sequencer *posr, list<Transaction*> &tls,
				  TrackedOpRef osd_op,
				  ThreadPool::TPHandle *handle)
{
	......
    journal->throttle();
	......
}

int FileJournal::prepare_multi_write(bufferlist& bl, uint64_t& orig_ops, uint64_t& orig_bytes)
{
	......
	put_throttle(1, peek_write().bl.length());
	......

}
```

## filestore apply

```cpp
int FileStore::queue_transactions(Sequencer *posr, list<Transaction*> &tls,
				  TrackedOpRef osd_op,
				  ThreadPool::TPHandle *handle)
{
	......
	op_queue_reserve_throttle(o, handle);
	......
}

void FileStore::_finish_op(OpSequencer *osr)
{
	......
    op_queue_release_throttle(o);
	......
}
```

## filestore writeback

wbthrottle，参见另外一篇[文章](http://blog.wjin.org/posts/ceph-throttle-summary.html)。

# Non-idempotent OP

在osd异常崩溃的情况下，journal中的数据不一定全部都存放在了FileStore的data disk中，因为apply到了FileStore中，并不代表数据就在disk中了，
此时很有可能数据在page cache中，需要sync线程调用fdatasync之类的系统调用才能保证数据落盘。

所以为了保证异常情况下数据的一致性，需要对journla的日志做回放，从什么地方开始回放，FileStore中会将已经apply到文件系统并进行
过fdatasync的序列号记录在文件commit\_op\_seq中，回放的时候就从此文件记录的序列号开始。

然而，回放的时候，部分op可能已经在disk中生效，但是commit\_op\_seq并没有体现，此时如果仍然回放，对于有些操作，反复执行多次会出问题，也即非幂等操作。

一个例子：

* clone一个object，这个操作已经提交到日志
* 将操作apply到FileStore也已经完成
* 源object后续做了更新，也apply到了FileStore

假设如上操作都已经体现在了disk中，但是sync线程并未来得及更新commit\_op\_seq，此时系统崩溃。
再次启动后，osd启动回放日志，第二次执行clone操作，将拷贝到新版本的数据，而不是期望版本的数据。
FileStore需要保证回放处理这种情况的正确性。

具体做法是在对象文件的属性中记录下最后操作的一个三元组（序列号，事务编号，OP编号），因为journal提交的时候有一个唯一的序列号，通过这个序列号，
就可以找到提交时候的事务，然后根据事务编号和OP编号最终定位出最后操作的OP。

```cpp
struct SequencerPosition {
  uint64_t seq;  ///< seq
  uint32_t trans; ///< transaction in that seq (0-based)
  uint32_t op;    ///< op in that transaction (0-based)
  ......
};
```

看clone例子，操作前，先检查下，如果可以继续执行，就执行操作，操作完成后，设置一个guard，这样对于非幂等操作，如果上次执行过，
肯定是有记录的，再一次执行的时候check就会失败，就不继续执行。

```cpp
int FileStore::_clone(const coll_t& cid, const ghobject_t& oldoid, const ghobject_t& newoid,
		      const SequencerPosition& spos)
{
  ......
  if (_check_replay_guard(cid, newoid, spos) < 0)
    return 0;

  ......

  // clone is non-idempotent; record our work.
  _set_replay_guard(**n, spos, &newoid);

  ......
}
```

# Tuning

```text
# journal 
journal_queue_max_bytes
journal_queue_max_ops

# filestore apply 
filestore_queue_max_bytes
filestore_queue_max_ops

# filestore writeback
filestore_wbthrottle_enable
filestore_wbthrottle_xfs_bytes_start_flusher
filestore_wbthrottle_xfs_bytes_hard_limit
filestore_wbthrottle_xfs_ios_start_flusher
filestore_wbthrottle_xfs_ios_hard_limit
filestore_wbthrottle_xfs_inodes_start_flusher
```

filestore writeback打开以后，一方面需要注意对应限流参数的调整，ext4和xfs是共用一套参数，
另一方面，如果io压力持续过大，可能导致FileStore::op\_tp被throttle住而超时，也有可能导致
FileStore::op\_wq限流起作用。

如果关闭writeback，可能导致FileStore:sync\_thread超时，需要调整参数filestore\_commit\_timeout，
ssd情况下可以关闭wbthrottle。


其他影响性能的参数：

```text
filestore_op_threads
filestore_fd_cache_size

journal_max_write_bytes
journal_max_write_entries

filestore_max_sync_interval
```
