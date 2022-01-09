---
title: "Nebula Storage 无法 Kill 问题分析"
date: 2022-01-09T16:48:52+08:00
draft: false
---

[Nebula storage](https://github.com/vesoft-inc/nebula) 实例经常出现无法 kill 的情况。
必须使用暴力的`kill -9`才能强行让它退出。
storage 无法 kill 一例：

```txt
# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   1104     4 ?        Ss   14:43   0:00 /sbin/docker-init -- /bin/bash -c /root/nebula-cluster/scripts/run-storage.sh
root           7  0.0  0.0   3976  3136 ?        S    14:43   0:00 /bin/bash /root/nebula-cluster/scripts/run-storage.sh
root          11  0.0  0.0  12176  4300 ?        Ss   14:43   0:00 sshd: /usr/sbin/sshd [listener] 0 of 10-100 startups
root          12 14.7  0.0 922936 82484 ?        Sl   14:43   3:39 /usr/local/bin/rr record -h -o /root/nebula-cluster/trace-002/store2 /root/src/nebula/build/bin/nebula-storaged --flagfile /root/nebula-cluster/conf/nebula-storaged.conf --raft_heartbeat_interval_secs 1 --meta_server
root          13  0.0  0.0   2508   588 ?        S    14:43   0:00 sleep infinity
root          25  1.1  0.0 901108 86144 ?        SLl  14:43   0:16 /root/src/nebula/build/bin/nebula-storaged --flagfile /root/nebula-cluster/conf/nebula-storaged.conf --raft_heartbeat_interval_secs 1 --meta_server_addrs=meta1:9559 --local_ip=store2 --data_path=data/store2 --pid_fil
root        3904  0.0  0.0  13580  8840 ?        Ds   15:06   0:00 sshd: root@pts/0
root        3915  0.0  0.0   4240  3512 pts/0    Ss   15:06   0:00 -bash
root        3925  0.0  0.0   5896  2896 pts/0    R+   15:08   0:00 ps aux
```

节点看起来似乎 hang 住了，查看日志，我们发现：

```txt
3485 I1207 15:06:29.026923    77 MetaClient.cpp:3034] Load leader ok
3486 I1207 15:06:29.356928    43 StorageDaemon.cpp:181] Signal 15(Terminated) received, stopping this server
3487 I1207 15:06:29.357704    43 NebulaStore.cpp:218] Stop the raft service...
3488 I1207 15:06:29.358178    43 RaftexService.cpp:125] Stopping the raftex service on port 9780
3489 I1207 15:06:29.358656    43 Host.cpp:43] [Port: 9780, Space: 1, Part: 1] [Host: store3:9780] The host has been stopped!
3490 I1207 15:06:29.359143    43 Host.cpp:43] [Port: 9780, Space: 1, Part: 1] [Host: store4:9780] The host has been stopped!
3491 I1207 15:06:29.359609    43 Host.cpp:43] [Port: 9780, Space: 1, Part: 1] [Host: store5:9780] The host has been stopped!
3492 I1207 15:06:29.360069    43 Host.cpp:43] [Port: 9780, Space: 1, Part: 1] [Host: store1:9780] The host has been stopped!
3493 I1207 15:06:29.360531    43 Host.h:32] [Port: 9780, Space: 1, Part: 1] [Host: store3:9780]  The host has been destroyed!
3494 I1207 15:06:29.361011    43 Host.h:32] [Port: 9780, Space: 1, Part: 1] [Host: store4:9780]  The host has been destroyed!
3495 I1207 15:06:29.361485    43 Host.h:32] [Port: 9780, Space: 1, Part: 1] [Host: store5:9780]  The host has been destroyed!
3496 I1207 15:06:29.361948    43 Host.h:32] [Port: 9780, Space: 1, Part: 1] [Host: store1:9780]  The host has been destroyed!
3497 I1207 15:06:29.362407    43 RaftPart.cpp:330] [Port: 9780, Space: 1, Part: 1] Partition has been stopped
3498 I1207 15:06:29.362891    43 RaftexService.cpp:132] All partitions have stopped
3499 I1207 15:06:29.371019    83 RaftexService.cpp:107] The Raftex Service stopped
3500 I1207 15:06:29.431828    43 AdminTaskManager.cpp:183] enter AdminTaskManager::shutdown()
3501 I1207 15:06:29.433558   122 AdminTaskManager.cpp:134] Unreported-Admin-Thread stopped
3502 I1207 15:06:29.438947   121 AdminTaskManager.cpp:238] detect AdminTaskManager::shutdown()
3503 I1207 15:06:29.442766    43 RocksEngine.h:119] Release rocksdb on data/store2/nebula/0
3504 I1207 15:06:29.454738    43 AdminTaskManager.cpp:200] exit AdminTaskManager::shutdown()
3505 I1207 15:06:29.475518    43 NebulaStore.cpp:36] Cut off the relationship with meta client
3506 I1207 15:06:29.476013    43 NebulaStore.cpp:39] Waiting for the raft service stop...
3507 I1207 15:33:26.880261    25 StorageDaemon.cpp:181] Signal 15(Terminated) received, stopping this server
3508 I1207 15:33:26.881636    25 StorageServer.cpp:321] All services has been stopped
```

从日志中我们看到 15:06:29 进程收到 kill 信号，
系统日志显示收到 SIGTERM 信号但是没有退出成功；
15:33:26 进程再次收到`kill -9`信号进程才退出。
注意到这样日志：3506 I1207 15:06:29.476013 43 NebulaStore.cpp:39] Waiting for the raft service stop...，难道是 raft service 卡住了？
翻下代码 NebulaStore.cpp：

```cpp
NebulaStore::~NebulaStore() {
  LOG(INFO) << "Cut off the relationship with meta client";
  options_.partMan_.reset();
  raftService_->stop();
  LOG(INFO) << "Waiting for the raft service stop...";
  raftService_->waitUntilStop();
  spaces_.clear();
  spaceListeners_.clear();
  bgWorkers_->stop();
  bgWorkers_->wait();
  storeWorker_->stop();
  storeWorker_->wait();
  LOG(INFO) << "~NebulaStore()";
}
```

猜测 storage 卡在四十行上：raftService_->waitUntilStop()。
由于我们用 Mozilla RR 把实例的运行过程录下来了，
所以开 RR debug，
continue 直接让把我们带到 SIGTERM 的位置，现场如下：

```txt

[New Thread 25.168]
--Type <RET> for more, q to quit, c to continue without paging--c
 
Thread 3 received signal SIGTERM, Terminated.
[Switching to Thread 25.43]
0x0000000070000002 in syscall_traced ()
(rr) bt
#0  0x0000000070000002 in syscall_traced ()
#1  0x0000250910a18725 in _raw_syscall () at /root/src/rr/src/preload/raw_syscall.S:120
#2  0x0000250910a144ff in traced_raw_syscall (call=call@entry=0x7f3c1c8fefa0) at /root/src/rr/src/preload/syscallbuf.c:278
#3  0x0000250910a15f98 in sys_statfs (call=<optimized out>) at /root/src/rr/src/preload/syscallbuf.c:3148
#4  syscall_hook_internal (call=0x7f3c1c8fefa0) at /root/src/rr/src/preload/syscallbuf.c:3415
#5  syscall_hook (call=0x7f3c1c8fefa0) at /root/src/rr/src/preload/syscallbuf.c:3454
#6  0x0000250910a14340 in _syscall_hook_trampoline () at /root/src/rr/src/preload/syscall_hook.S:313
#7  0x0000250910a1439f in __morestack () at /root/src/rr/src/preload/syscall_hook.S:458
#8  0x0000250910a143bb in _syscall_hook_trampoline_48_3d_00_f0_ff_ff () at /root/src/rr/src/preload/syscall_hook.S:477
#9  0x00002509109f92d5 in __libc_write (nbytes=8, buf=0x2c941f48e08, fd=31) at ../sysdeps/unix/sysv/linux/write.c:26
#10 __libc_write (fd=31, buf=0x2c941f48e08, nbytes=8) at ../sysdeps/unix/sysv/linux/write.c:24
#11 0x000000000496e61e in folly::EventBaseAtomicNotificationQueue<folly::Function<void ()>, folly::EventBase::FuncRunner>::notifyFd() ()
#12 0x000000000496829e in folly::EventBase::runInEventBaseThread(folly::Function<void ()>) ()
#13 0x00000000045a922b in apache::thrift::HandlerCallbackBase::sendReply(folly::IOBufQueue) ()
```

SIGTERM 断点的地方看不到什么有用的信息，继续 continue，在我们给 NebulaStore.cpp 设置的断点中断下来了：

```txt
(rr) c
Continuing.
 
Thread 3 hit Breakpoint 1, nebula::kvstore::NebulaStore::~NebulaStore (this=0x250910837400, __in_chrg=<optimized out>) at /root/src/nebula/src/kvstore/NebulaStore.cpp:40
40    raftService_->waitUntilStop();
(rr)
```

断是断下来了，但是看不出什么有用的信息。

我是灵感的分割线
____________

我们给 storage 实例发了两次 SIGTERM 信号。
rr 调试中在每次收到 SIGTERM 信号时都会断下来。
第一次断下来的时候，
storage 上大部分线程都还是运行状态，
第二次断下来则有相当部分的线程已经退出了，
这其中就包括导致 storage 无法退出的线程。
通过堆栈， 我可以确认 ThriftServer 在退出时卡死了。
ThriftServer 显然是在等待另一个线程的退出。
我们高度怀疑这个真正退不出的线程其实是 ThriftServer 管理着的一个 worker 线程退不出。
还是通过堆栈，我们在每次 storage 卡死的时候都能看到一个似乎是在跑 appendLog 任务的 thrift worker 线程，
这个 worker 运行的 appendLog 任务卡死可能是 storage 无法退出的本质原因。
第一次收到 SIGTERM，我们观察到此时系统中的 Thrift executor 大概有 40 个左右：

```txt
Thread 3 received signal SIGTERM, Terminated.
[Switching to Thread 25.43]
0x0000000070000002 in syscall_traced ()
(rr) info threads
  Id   Target Id                       Frame
  1    Thread 25.25 (nebula-storaged)  __pthread_clockjoin_ex (threadid=11519218898688, thread_return=0x0, clockid=<optimized out>, abstime=<optimized out>, block=<optimized out>) at pthread_join_common.c:145
  2    Thread 25.66 (IOThreadPool0)    0x0000000070000002 in syscall_traced ()
* 3    Thread 25.43 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  4    Thread 25.26 (executor-pri0-1)  0x0000000070000002 in syscall_traced ()
  5    Thread 25.27 (executor-pri0-2)  0x0000000070000002 in syscall_traced ()
  6    Thread 25.28 (executor-pri1-1)  0x0000000070000002 in syscall_traced ()
  7    Thread 25.29 (executor-pri1-2)  0x0000000070000002 in syscall_traced ()
  8    Thread 25.30 (executor-pri2-1)  0x0000000070000002 in syscall_traced ()
  9    Thread 25.31 (executor-pri2-2)  0x0000000070000002 in syscall_traced ()
  10   Thread 25.32 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  11   Thread 25.33 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  12   Thread 25.34 (executor-pri3-5)  0x0000000070000002 in syscall_traced ()
  13   Thread 25.35 (executor-pri3-3)  0x0000000070000002 in syscall_traced ()
  14   Thread 25.36 (executor-pri3-4)  0x0000000070000002 in syscall_traced ()
  15   Thread 25.37 (executor-pri3-6)  0x0000000070000002 in syscall_traced ()
  16   Thread 25.38 (executor-pri3-7)  0x0000000070000002 in syscall_traced ()
  17   Thread 25.39 (executor-pri3-8)  0x0000000070000002 in syscall_traced ()
  18   Thread 25.40 (executor-pri3-9)  0x0000000070000002 in syscall_traced ()
  19   Thread 25.41 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  20   Thread 25.42 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  21   Thread 25.44 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  22   Thread 25.45 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  23   Thread 25.46 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  24   Thread 25.47 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  25   Thread 25.48 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  26   Thread 25.49 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  27   Thread 25.50 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  28   Thread 25.51 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  29   Thread 25.52 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  30   Thread 25.53 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  31   Thread 25.54 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  32   Thread 25.55 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  33   Thread 25.56 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  34   Thread 25.57 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  35   Thread 25.58 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  36   Thread 25.59 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  37   Thread 25.60 (executor-pri3-2)  0x0000000070000002 in syscall_traced ()
  38   Thread 25.61 (executor-pri3-3)  0x0000000070000002 in syscall_traced ()
  39   Thread 25.62 (executor-pri3-3)  0x0000000070000002 in syscall_traced ()
  40   Thread 25.63 (executor-pri3-3)  0x0000000070000002 in syscall_traced ()
  41   Thread 25.64 (executor-pri4-1)  0x0000000070000002 in syscall_traced ()
  42   Thread 25.65 (executor-pri4-2)  0x0000000070000002 in syscall_traced ()
  43   Thread 25.67 (IOThreadPool1)    0x0000000070000002 in syscall_traced ()
  44   Thread 25.68 (IOThreadPool2)    0x0000000070000002 in syscall_traced ()
  45   Thread 25.69 (tsc-calibrator)   0x00005d55579843bf in __GI___clock_nanosleep (clock_id=clock_id@entry=0, flags=flags@entry=0, req=req@entry=0x7bec2206c9f0, rem=rem@entry=0x7bec2206c9f0) at ../sysdeps/unix/sysv/linux/clock_nanosleep.c:78
  46   Thread 25.70 (IOThreadPool3)    0x0000000070000002 in syscall_traced ()
  47   Thread 25.71 (IOThreadPool4)    0x0000000070000002 in syscall_traced ()
  48   Thread 25.72 (IOThreadPool5)    0x0000000070000002 in syscall_traced ()
```

第二次只剩下三个：

```txt

Thread 1 received signal SIGTERM, Terminated.
[Switching to Thread 25.25]
__pthread_clockjoin_ex (threadid=11519218898688, thread_return=0x0, clockid=<optimized out>, abstime=<optimized out>, block=<optimized out>) at pthread_join_common.c:145
145 pthread_join_common.c: No such file or directory.
(rr) info threads
(rr) info threads
  Id   Target Id                       Frame
* 1    Thread 25.25 (nebula-storaged)  __pthread_clockjoin_ex (threadid=11519218898688, thread_return=0x0, clockid=<optimized out>, abstime=<optimized out>, block=<optimized out>) at pthread_join_common.c:145
  2    Thread 25.66 (IOThreadPool0)    0x0000000070000002 in syscall_traced ()
  3    Thread 25.43 (executor-pri3-1)  0x0000000070000002 in syscall_traced ()
  41   Thread 25.64 (executor-pri4-1)  0x0000000070000002 in syscall_traced ()
  42   Thread 25.65 (executor-pri4-2)  0x0000000070000002 in syscall_traced ()
  43   Thread 25.67 (IOThreadPool1)    0x0000000070000002 in syscall_traced ()
  44   Thread 25.68 (IOThreadPool2)    0x0000000070000002 in syscall_traced ()
  45   Thread 25.69 (tsc-calibrator)   0x00005d55579843bf in __GI___clock_nanosleep (clock_id=clock_id@entry=0, flags=flags@entry=0, req=req@entry=0x7bec2206c9f0, rem=rem@entry=0x7bec2206c9f0) at ../sysdeps/unix/sysv/linux/clock_nanosleep.c:78
  46   Thread 25.70 (IOThreadPool3)    0x0000000070000002 in syscall_traced ()
  47   Thread 25.71 (IOThreadPool4)    0x0000000070000002 in syscall_traced ()
  48   Thread 25.72 (IOThreadPool5)    0x0000000070000002 in syscall_traced ()
  49   Thread 25.73 (IOThreadPool6)    0x0000000070000002 in syscall_traced ()
  50   Thread 25.74 (IOThreadPool7)    0x0000000070000002 in syscall_traced ()
  51   Thread 25.75 (IOThreadPool8)    0x0000000070000002 in syscall_traced ()
  52   Thread 25.76 (IOThreadPool9)    0x0000000070000002 in syscall_traced ()
  54   Thread 25.78 (nebula-bgworker)  0x0000000070000002 in syscall_traced ()
  55   Thread 25.79 (nebula-bgworker)  0x0000000070000002 in syscall_traced ()
  56   Thread 25.80 (nebula-bgworker)  0x0000000070000002 in syscall_traced ()
  57   Thread 25.81 (nebula-bgworker)  0x0000000070000002 in syscall_traced ()
```

检查 thread 3，看到一个非常好玩的堆栈：

```txt
(rr) bt
#0  0x0000000070000002 in syscall_traced ()
#1  0x0000250910a18725 in _raw_syscall () at /root/src/rr/src/preload/raw_syscall.S:120
#2  0x0000250910a144ff in traced_raw_syscall (call=call@entry=0x7f3c1c8fd970) at /root/src/rr/src/preload/syscallbuf.c:278
#3  0x0000250910a15f98 in sys_statfs (call=<optimized out>) at /root/src/rr/src/preload/syscallbuf.c:3148
#4  syscall_hook_internal (call=0x7f3c1c8fd970) at /root/src/rr/src/preload/syscallbuf.c:3415
#5  syscall_hook (call=0x7f3c1c8fd970) at /root/src/rr/src/preload/syscallbuf.c:3454
#6  0x0000250910a14340 in _syscall_hook_trampoline () at /root/src/rr/src/preload/syscall_hook.S:313
#7  0x0000250910a1439f in __morestack () at /root/src/rr/src/preload/syscall_hook.S:458
#8  0x0000250910a143bb in _syscall_hook_trampoline_48_3d_00_f0_ff_ff () at /root/src/rr/src/preload/syscall_hook.S:477
#9  0x00002509109f537c in futex_wait_cancelable (private=<optimized out>, expected=0, futex_word=0x250910833a2c) at ../sysdeps/nptl/futex-internal.h:183
#10 __pthread_cond_wait_common (abstime=0x0, clockid=0, mutex=0x250910833930, cond=0x250910833a00) at pthread_cond_wait.c:508
#11 __pthread_cond_wait (cond=0x250910833a00, mutex=0x250910833930) at pthread_cond_wait.c:638
#12 0x0000000004ed23e0 in std::condition_variable::wait(std::unique_lock<std::mutex>&) ()
#13 0x00000000046d867b in apache::thrift::concurrency::ThreadManager::Impl::removeWorkerImpl(std::unique_lock<std::mutex>&, unsigned long, bool) ()
#14 0x00000000046d9c1b in apache::thrift::concurrency::ThreadManager::Impl::stopImpl(bool) ()
#15 0x00000000046e09f9 in apache::thrift::concurrency::PriorityThreadManager::PriorityImpl::join() ()
#16 0x00000000045deee7 in apache::thrift::ThriftServer::~ThriftServer() ()
#17 0x00000000045df3e2 in apache::thrift::ThriftServer::~ThriftServer() ()
#18 0x0000000002be4223 in std::default_delete<apache::thrift::ThriftServer>::operator() (this=0x250910812ac0, __ptr=0x25091081e000) at /usr/include/c++/9/bits/unique_ptr.h:81
#19 0x0000000002be7100 in std::unique_ptr<apache::thrift::ThriftServer, std::default_delete<apache::thrift::ThriftServer> >::reset (this=0x250910812ac0, __p=0x25091081e000) at /usr/include/c++/9/bits/unique_ptr.h:402
#20 0x0000000004110b8a in nebula::raftex::RaftexService::waitUntilStop (this=0x250910812ab0) at /root/src/nebula/src/kvstore/raftex/RaftexService.cpp:141
#21 0x00000000040652a2 in nebula::kvstore::NebulaStore::~NebulaStore (this=0x250910837400, __in_chrg=<optimized out>) at /root/src/nebula/src/kvstore/NebulaStore.cpp:40
#22 0x00000000040654fe in nebula::kvstore::NebulaStore::~NebulaStore (this=0x250910837400, __in_chrg=<optimized out>) at /root/src/nebula/src/kvstore/NebulaStore.cpp:48
#23 0x0000000002be442f in std::default_delete<nebula::kvstore::KVStore>::operator() (this=0x25091082f670, __ptr=0x250910837400) at /usr/include/c++/9/bits/unique_ptr.h:81
#24 0x0000000002bdf1b8 in std::unique_ptr<nebula::kvstore::KVStore, std::default_delete<nebula::kvstore::KVStore> >::reset (this=0x25091082f670, __p=0x250910837400) at /usr/include/c++/9/bits/unique_ptr.h:402
#25 0x0000000002bd76f7 in nebula::storage::StorageServer::stop (this=0x25091082f600) at /root/src/nebula/src/storage/StorageServer.cpp:351
#26 0x0000000002bbaf2d in signalHandler (sig=15) at /root/src/nebula/src/daemons/StorageDaemon.cpp:183
#27 0x0000000002bbad2f in <lambda(nebula::SignalHandler::GeneralSignalInfo*)>::operator()(nebula::SignalHandler::GeneralSignalInfo *) const (__closure=0x5d93360 <nebula::SignalHandler::get()::instance+448>, info=0x7f3c1c8fe030) at /root/src/nebula/src/daemons/StorageDaemon.cpp:174
#28 0x0000000002bc2ad8 in std::_Function_handler<void(nebula::SignalHandler::GeneralSignalInfo*), setupSignalHandler()::<lambda(nebula::SignalHandler::GeneralSignalInfo*)> >::_M_invoke(const std::_Any_data &, nebula::SignalHandler::GeneralSignalInfo *&&) (__functor=...,
    __args#0=@0x7f3c1c8fdfe0: 0x7f3c1c8fe030) at /usr/include/c++/9/bits/std_function.h:300
#29 0x0000000003f5abc8 in std::function<void (nebula::SignalHandler::GeneralSignalInfo*)>::operator()(nebula::SignalHandler::GeneralSignalInfo*) const (this=0x5d93360 <nebula::SignalHandler::get()::instance+448>, __args#0=0x7f3c1c8fe030)
    at /usr/include/c++/9/bits/std_function.h:688
#30 0x0000000003f5a6e1 in nebula::SignalHandler::handleGeneralSignal (this=0x5d931a0 <nebula::SignalHandler::get()::instance>, sig=15, info=0x7f3c1c8fe1f0) at /root/src/nebula/src/common/base/SignalHandler.cpp:106
#31 0x0000000003f5a669 in nebula::SignalHandler::doHandle (this=0x5d931a0 <nebula::SignalHandler::get()::instance>, sig=15, info=0x7f3c1c8fe1f0, uctx=0x7f3c1c8fe0c0) at /root/src/nebula/src/common/base/SignalHandler.cpp:100
#32 0x0000000003f5a5c5 in nebula::SignalHandler::handlerHook (sig=15, info=0x7f3c1c8fe1f0, uctx=0x7f3c1c8fe0c0) at /root/src/nebula/src/common/base/SignalHandler.cpp:81
#33 <signal handler called>
#34 0x0000000070000002 in syscall_traced ()
#35 0x0000250910a18725 in _raw_syscall () at /root/src/rr/src/preload/raw_syscall.S:120
#36 0x0000250910a144ff in traced_raw_syscall (call=call@entry=0x7f3c1c8fefa0) at /root/src/rr/src/preload/syscallbuf.c:278
#37 0x0000250910a15f98 in sys_statfs (call=<optimized out>) at /root/src/rr/src/preload/syscallbuf.c:3148
#38 syscall_hook_internal (call=0x7f3c1c8fefa0) at /root/src/rr/src/preload/syscallbuf.c:3415
#39 syscall_hook (call=0x7f3c1c8fefa0) at /root/src/rr/src/preload/syscallbuf.c:3454
#40 0x0000250910a14340 in _syscall_hook_trampoline () at /root/src/rr/src/preload/syscall_hook.S:313
#41 0x0000250910a1439f in __morestack () at /root/src/rr/src/preload/syscall_hook.S:458
#42 0x0000250910a143bb in _syscall_hook_trampoline_48_3d_00_f0_ff_ff () at /root/src/rr/src/preload/syscall_hook.S:477
#43 0x00002509109f92d5 in __libc_write (nbytes=8, buf=0x2c941f48e08, fd=31) at ../sysdeps/unix/sysv/linux/write.c:26
#44 __libc_write (fd=31, buf=0x2c941f48e08, nbytes=8) at ../sysdeps/unix/sysv/linux/write.c:24
#45 0x000000000496e61e in folly::EventBaseAtomicNotificationQueue<folly::Function<void ()>, folly::EventBase::FuncRunner>::notifyFd() ()
#46 0x000000000496829e in folly::EventBase::runInEventBaseThread(folly::Function<void ()>) ()
#47 0x00000000045a922b in apache::thrift::HandlerCallbackBase::sendReply(folly::IOBufQueue) ()
#48 0x00000000041e3581 in apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse>::doResult (this=0x241a228b6a80, r=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/async/AsyncProcessor.h:865
#49 0x00000000041dd581 in apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse>::result (this=0x241a228b6a80, r=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/async/AsyncProcessor.h:568
#50 0x00000000041d76d6 in apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse>::complete (this=0x241a228b6a80, r=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/async/AsyncProcessor.h:851
#51 0x00000000041d44c9 in apache::thrift::detail::si::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>::operator()(folly::Try<nebula::raftex::cpp2::AppendLogResponse> &&) const (this=0x14613bee3d10, _ret=...)
    at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1265
#52 0x00000000041d7753 in folly::Future<nebula::raftex::cpp2::AppendLogResponse>::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>::operator()(folly::Executor::KeepAlive<folly::Executor> &&, folly::Try<nebula::raftex::cpp2::AppendLogResponse> &&) (this=0x14613bee3d10, t=...) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:961
#53 0x00000000041dd63d in folly::futures::detail::CoreCallbackState<folly::Unit, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>; apache::thrift::detail::si::CallbackPtr<F> = std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >; typename folly::drop_unit<typename folly::invoke_detail::traits<F>::result<>::value_type>::type = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse; typename folly::futures::detail::tryCallableResult<T, F>::value_type = folly::Unit]::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> >::invoke<folly::Executor::KeepAlive<folly::Executor>, folly::Try<nebula::raftex::cpp2::AppendLogResponse> >(void) (this=0x14613bee3d10) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:134
#54 0x00000000041dd696 in folly::futures::detail::detail_msvc_15_7_workaround::invoke<folly::futures::detail::tryExecutorCallableResult<nebula::raftex::cpp2::AppendLogResponse, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::Ser--Type <RET> for more, q to quit, c to continue without paging--
verInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, void>, folly::futures::detail::CoreCallbackState<folly::Unit, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> >, nebula::raftex::cpp2::AppendLogResponse>(folly::futures::detail::tryExecutorCallableResult<nebula::raftex::cpp2::AppendLogResponse, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>; apache::thrift::detail::si::CallbackPtr<F> = std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >; typename folly::drop_unit<typename folly::invoke_detail::traits<F>::result<>::value_type>::type = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse; typename folly::futures::detail::tryCallableResult<T, F>::value_type = folly::Unit]::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, void>, folly::futures::detail::CoreCallbackState<folly::Unit, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>; apache::thrift::detail::si::CallbackPtr<F> = std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >; typename folly::drop_unit<typename folly::invoke_detail::traits<F>::result<>::value_type>::type = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse; typename folly::futures::detail::tryCallableResult<T, F>::value_type = folly::Unit]::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> > &, folly::Executor::KeepAlive<folly::Executor> &&, folly::Try<nebula::raftex::cpp2::AppendLogResponse> &&) (state=..., ka=..., t=...) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:327
#55 0x00000000041dd6e4 in folly::futures::detail::FutureBase<nebula::raftex::cpp2::AppendLogResponse>::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>::<lambda()>::operator()(void) const (this=0x2c941f49240)
    at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:402
#56 0x00000000041e9555 in folly::makeTryWithNoUnwrap<folly::futures::detail::FutureBase<T>::thenImplementation(F&&, R, folly::futures::detail::InlineContinuation) [with F = folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; R = folly::futures::detail::tryExecutorCallableResult<nebula::raftex::cpp2::AppendLogResponse, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, void>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> mutable::<lambda()> >(folly::futures::detail::FutureBase<nebula::raftex::cpp2::AppendLogResponse>::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>::<lambda()> &&) (f=...) at /root/src/nebula/build/third-party/install/include/folly/Try-inl.h:257
#57 0x00000000041e3880 in folly::makeTryWith<folly::futures::detail::FutureBase<T>::thenImplementation(F&&, R, folly::futures::detail::InlineContinuation) [with F = folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; R = folly::futures::detail::tryExecutorCallableResult<nebula::raftex::cpp2::AppendLogResponse, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, void>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> mutable::<lambda()> >(folly::futures::detail::FutureBase<nebula::raftex::cpp2::AppendLogResponse>::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>::<lambda()> &&) (f=...) at /root/src/nebula/build/third-party/install/include/folly/Try-inl.h:270
#58 0x00000000041dd7ce in folly::futures::detail::FutureBase<nebula::raftex::cpp2::AppendLogResponse>::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>::operator()(folly::Executor::KeepAlive<folly::Executor> &&, folly::Try<nebula::raftex::cpp2::AppendLogResponse> &&) (this=0x14613bee3d10, ka=..., t=...) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:401
#59 0x00000000041e9738 in folly::futures::detail::Core<nebula::raftex::cpp2::AppendLogResponse>::<lambda(folly::futures::detail::CoreBase&, folly::Executor::KeepAlive<folly::Executor>&&, folly::exception_wrapper*)>::operator()(folly::futures::detail::CoreBase &, folly::Executor::KeepAlive<folly::Executor> &&, folly::exception_wrapper *) (this=0x14613bee3d10, coreBase=..., ka=..., ew=0x0) at /root/src/nebula/build/third-party/install/include/folly/futures/detail/Core.h:583
#60 0x00000000041f21ec in folly::detail::function::FunctionTraits<void(folly::futures::detail::CoreBase&, folly::Executor::KeepAlive<folly::Executor>&&, folly::exception_wrapper*)>::callSmall<folly::futures::detail::Core<T>::setCallback(F&&, std::shared_ptr<folly::RequestContext>&&, folly::futures::detail::InlineContinuation) [with F = folly::futures::detail::FutureBase<T>::thenImplementation(F&&, R, folly::futures::detail::InlineContinuation) [with F = folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; R = folly::futures::detail::tryExecutorCallableResult<nebula::raftex::cpp2::AppendLogResponse, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, void>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::futures::detail::CoreBase&, folly::Executor::KeepAlive<>&&, folly::exception_wrapper*)> >(folly::detail::function::CallArg, folly::detail::function::CallArg, folly::detail::function::CallArg, folly::detail::function::Data &) (args#0=..., args#1=..., args#2=0x0, p=...)
    at /root/src/nebula/build/third-party/install/include/folly/Function.h:371
#61 0x000000000492ac9c in ?? ()
#62 0x000000000492ba84 in folly::futures::detail::CoreBase::doCallback(folly::Executor::KeepAlive<folly::Executor>&&, folly::futures::detail::State) ()
#63 0x000000000492bfbf in folly::futures::detail::CoreBase::setCallback_(folly::Function<void (folly::futures::detail::CoreBase&, folly::Executor::KeepAlive<folly::Executor>&&, folly::exception_wrapper*)>&&, std::shared_ptr<folly::RequestContext>&&, folly::futures::detail::InlineContinuation) ()
#64 0x00000000041e983c in folly::futures::detail::Core<nebula::raftex::cpp2::AppendLogResponse>::setCallback<folly::futures::detail::FutureBase<T>::thenImplementation(F&&, R, folly::futures::detail::InlineContinuation) [with F = folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; R = folly::futures::detail::tryExecutorCallableResult<nebula::raftex::cpp2::AppendLogResponse, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, void>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> >(folly::futures::detail::FutureBase<nebula::raftex::cpp2::AppendLogResponse>::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> &&, std::shared_ptr<folly::RequestContext> &&, folly::futures::detail::InlineContinuation) (this=0x14613bee3d00,
    func=..., context=..., allowInline=folly::futures::detail::InlineContinuation::permit) at /root/src/nebula/build/third-party/install/include/folly/futures/detail/Core.h:586
#65 0x00000000041e39c0 in folly::futures::detail::FutureBase<nebula::raftex::cpp2::AppendLogResponse>::setCallback_<folly::futures::detail::FutureBase<T>::thenImplementation(F&&, R, folly::futures::detail::InlineContinuation) [with F = folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; R = folly::futures::detail::tryExecutorCallableResult<nebula::raftex::cpp2::AppendLogResponse, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, void>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> >(folly::futures::detail::FutureBase<nebula::raftex::cpp2::AppendLogResponse>::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> &&, folly::futures::detail::InlineContinuation) (this=0x2c941f49748, func=...,
    allowInline=folly::futures::detail::InlineContinuation::permit) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:303
#66 0x00000000041dd9de in folly::futures::detail::FutureBase<nebula::raftex::cpp2::AppendLogResponse>::thenImplementation<folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<--Type <RET> for more, q to quit, c to continue without paging--
F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, folly::futures::detail::tryExecutorCallableResult<nebula::raftex::cpp2::AppendLogResponse, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Executor::KeepAlive<>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, void> >(folly::Future<nebula::raftex::cpp2::AppendLogResponse>::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> &&, folly::futures::detail::tryExecutorCallableResult<nebula::raftex::cpp2::AppendLogResponse, folly::Future<T>::thenTryInline(F&&) && [with F = apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>; apache::thrift::detail::si::CallbackPtr<F> = std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >; typename folly::drop_unit<typename folly::invoke_detail::traits<F>::result<>::value_type>::type = nebula::raftex::cpp2::AppendLogResponse]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>; T = nebula::raftex::cpp2::AppendLogResponse; typename folly::futures::detail::tryCallableResult<T, F>::value_type = folly::Unit]::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)>, void>, folly::futures::detail::InlineContinuation) (this=0x2c941f49748, func=..., allowInline=folly::futures::detail::InlineContinuation::permit)
    at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:393
#67 0x00000000041d781b in folly::Future<nebula::raftex::cpp2::AppendLogResponse>::thenTryInline<apache::thrift::detail::si::async_tm(apache::thrift::ServerInterface*, apache::thrift::detail::si::CallbackPtr<F>, F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()>]::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> >(apache::thrift::detail::si::<lambda(folly::Try<nebula::raftex::cpp2::AppendLogResponse>&&)> &&) (this=0x2c941f49748, func=...) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:965
#68 0x00000000041d45b0 in apache::thrift::detail::si::async_tm<nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()> >(apache::thrift::ServerInterface *, apache::thrift::detail::si::CallbackPtr, nebula::raftex::cpp2::RaftexServiceSvIf::<lambda()> &&) (si=0x250910812ab8, callback=..., f=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1263
#69 0x00000000041d2acd in nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog (this=0x250910812ab0, callback=..., p_req=...) at /root/src/nebula/build/src/interface/gen-cpp2/RaftexService.cpp:55
#70 0x00000000041dbbf3 in nebula::raftex::cpp2::RaftexServiceAsyncProcessor::process_appendLog<apache::thrift::CompactProtocolReader, apache::thrift::CompactProtocolWriter> (this=0x38aa18e3c460, req=..., serializedRequest=..., ctx=0x38aa18e19f10, eb=0x38aa18dc7000,
    tm=0x2509107fa1d0) at /root/src/nebula/build/src/interface/gen-cpp2/RaftexService.tcc:113
#71 0x00000000041df85c in apache::thrift::GeneratedAsyncProcessor::makeEventTaskForRequest<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::raftex::cpp2::RaftexServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::raftex::cpp2::RaftexServiceAsyncProcessor*, apache::thrift::Tile*)::{lambda(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>)#1}::operator()(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>) (this=0x45fc3a8eec00, rq=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/async/AsyncProcessor.h:698
#72 0x00000000041f74d2 in folly::detail::function::FunctionTraits<void (std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>)>::callBig<apache::thrift::GeneratedAsyncProcessor::makeEventTaskForRequest<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::raftex::cpp2::RaftexServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::raftex::cpp2::RaftexServiceAsyncProcessor*, apache::thrift::Tile*)::{lambda(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>)#1}>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>&&, folly::detail::function::Data&) (args#0=..., p=...) at /root/src/nebula/build/third-party/install/include/folly/Function.h:385
#73 0x00000000045a8fa2 in apache::thrift::EventTask::run() ()
#74 0x00000000041d9bdf in apache::thrift::GeneratedAsyncProcessor::processInThread<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::raftex::cpp2::RaftexServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::raftex::cpp2::RaftexServiceAsyncProcessor*)::{lambda()#1}::operator()() const (this=0x14613bea4490)
    at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/async/AsyncProcessor.h:773
#75 0x00000000041eac5e in folly::detail::function::FunctionTraits<void ()>::callSmall<apache::thrift::GeneratedAsyncProcessor::processInThread<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::raftex::cpp2::RaftexServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::raftex::cpp2::RaftexServiceAsyncProcessor*)::{lambda()#1}>(folly::detail::function::Data&) (p=...) at /root/src/nebula/build/third-party/install/include/folly/Function.h:371
#76 0x00000000045ae447 in virtual thunk to apache::thrift::concurrency::FunctionRunner::run() ()
#77 0x00000000046e3843 in apache::thrift::concurrency::ThreadManager::Impl::Worker::run() ()
#78 0x00000000046e749d in apache::thrift::concurrency::PthreadThread::threadMain(void*) ()
#79 0x00002509109ee609 in start_thread (arg=<optimized out>) at pthread_create.c:477
#80 0x00005d55579c6293 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```

首先这是个 worker 线程，
从堆栈底部可以看出这个线程正在执行 appendLog 相关的操作。
然后系统收到 SIGTERM 信号了，
信号处理的 handler 被调度到这个 worker 所在的线程上执行，
这个信号处理程序都干了些啥呢？
他调用 StorageDaemon.cpp 中的 signalHandler() 停止 StorageServer：

```cpp
void signalHandler(int sig) {
  switch (sig) {
    case SIGINT:
    case SIGTERM:
      FLOG_INFO("Signal %d(%s) received, stopping this server", sig, ::strsignal(sig));
      if (gStorageServer) {
        gStorageServer->stop();
      }
      break;
    default:
      FLOG_ERROR("Signal %d(%s) received but ignored", sig, ::strsignal(sig));
  }
}
```

然后 StorageServer.cpp 调用`StorageServer::stop()`停止 StorageServer，
StorageServer 会尝试停止他持有的 ThriftServer，
ThriftServer 在停止前会等待他的所有 worker 结束，
可以从以上堆栈的顶部的函数调用看出这点：

```txt
#12 0x0000000004ed23e0 in std::condition_variable::wait(std::unique_lock<std::mutex>&) ()
#13 0x00000000046d867b in apache::thrift::concurrency::ThreadManager::Impl::removeWorkerImpl(std::unique_lock<std::mutex>&, unsigned long, bool) ()
#14 0x00000000046d9c1b in apache::thrift::concurrency::ThreadManager::Impl::stopImpl(bool) ()
#15 0x00000000046e09f9 in apache::thrift::concurrency::PriorityThreadManager::PriorityImpl::join() ()
#16 0x00000000045deee7 in apache::thrift::ThriftServer::~ThriftServer() ()
#17 0x00000000045df3e2 in apache::thrift::ThriftServer::~ThriftServer() ()
#18 0x0000000002be4223 in std::default_delete<apache::thrift::ThriftServer>::operator() (this=0x250910812ac0, __ptr=0x25091081e000) at /usr/include/c++/9/bits/unique_ptr.h:81
#19 0x0000000002be7100 in std::unique_ptr<apache::thrift::ThriftServer, std::default_delete<apache::thrift::ThriftServer> >::reset (this=0x250910812ac0, __p=0x25091081e000) at /usr/include/c++/9/bits/unique_ptr.h:402
#20 0x0000000004110b8a in nebula::raftex::RaftexService::waitUntilStop (this=0x250910812ab0) at /root/src/nebula/src/kvstore/raftex/RaftexService.cpp:141
#21 0x00000000040652a2 in nebula::kvstore::NebulaStore::~NebulaStore (this=0x250910837400, __in_chrg=<optimized out>) at /root/src/nebula/src/kvstore/NebulaStore.cpp:40
#22 0x00000000040654fe in nebula::kvstore::NebulaStore::~NebulaStore (this=0x250910837400, __in_chrg=<optimized out>) at /root/src/nebula/src/kvstore/NebulaStore.cpp:48
#23 0x0000000002be442f in std::default_delete<nebula::kvstore::KVStore>::operator() (this=0x25091082f670, __ptr=0x250910837400) at /usr/include/c++/9/bits/unique_ptr.h:81
#24 0x0000000002bdf1b8 in std::unique_ptr<nebula::kvstore::KVStore, std::default_delete<nebula::kvstore::KVStore> >::reset (this=0x25091082f670, __p=0x250910837400) at /usr/include/c++/9/bits/unique_ptr.h:402
#25 0x0000000002bd76f7 in nebula::storage::StorageServer::stop (this=0x25091082f600) at /root/src/nebula/src/storage/StorageServer.cpp:351
#26 0x0000000002bbaf2d in signalHandler (sig=15) at /root/src/nebula/src/daemons/StorageDaemon.cpp:183
```

所以这里是不有个死循环？
信号处理 handler 在 worker 上运行，
信号处理 handler 等待 storage 结束，
storage 等待 ThriftServer 结束，
ThriftServer 等待 worker 结束，首尾衔接妙不可言的死循环啊……

所以，为什么信号处理在分发到这个线程呢？翻 Linux signal() 函数的手册我们发现，系统是随机挑选线程的，会不会踩坑主要看运气了：

![signal thread](/images/ns-kill/signal-thread.jpg)

顺便贴一张 Linux 信号处理回调的运行流程：

![signal exec](/images/ns-kill/signal-handler-exec.jpg)

这个问题如何处理应该非常明确了，
原则上，signal handler 里面不应该做太重的操作，他应该尽开的返回，
wait()、join() 这样的调用尽量不要放在 signal handler 里面（应该说这种行为就得杜绝）。
