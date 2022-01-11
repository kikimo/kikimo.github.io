---
title: "Nebula Storage 疯了（一）"
date: 2022-01-11T21:05:11+08:00
draft: false
---

问题详情参考这两个 issue：

1. Storage go crazy while balancing data #3664
2. Extra parts found after performing balance operation #3657

这个问题概括而言就是：
在 storage balance data 的时候如果触发 leader change 那么集群中至少会发现有一个节点进入一种疯狂的状态——CPU 飙升到 3000+%，
pstack 显示有大量线程在获取 RWSpinlock。

```txt
# pidstat -p 14 -u 1
Linux 5.4.0-91-generic (store1)     01/11/22    _x86_64_    (128 CPU)
 
10:25:23      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
10:25:24        0        14 3147.00 1755.00    0.00    0.00 4902.00    27  nebula-storaged
10:25:25        0        14 4899.00    0.00    0.00    0.00 4899.00    27  nebula-storaged
10:25:26        0        14 3979.00  921.00    0.00    0.00 4900.00    27  nebula-storaged
10:25:27        0        14 3095.00 1765.00    0.00    0.00 4860.00    27  nebula-storaged
10:25:28        0        14 4900.00    0.00    0.00    0.00 4900.00    27  nebula-storaged
10:25:29        0        14 4020.00  878.00    0.00    0.00 4898.00    27  nebula-storaged
10:25:30        0        14 3089.00 1810.00    0.00    0.00 4899.00    27  nebula-storaged
^C
Average:        0        14 3875.57 1018.43    0.00    0.00 4894.00     -  nebula-storaged
```

猜测主要问题应该就在 spinlock 上。
观察 pstack 中的堆栈，所有的 spinlock 似乎都是 RaftexService.cpp:165 这样代码调用的：

```txt

thread: 0, lwp: 22, type: 0
#0  0x000000000352adeb in folly::RWSpinLock::try_lock_shared()+107 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RWSpinLock.h:281
#1  0x000000000352ac41 in folly::RWSpinLock::lock_shared()+34 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RWSpinLock.h:210
#2  0x000000000352ae65 in ReadHolder()+44 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RWSpinLock.h:320
#3  0x00000000048819fd in nebula::raftex::RaftexService::findPart()+68 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RaftexService.cpp:165
#4  0x0000000004882067 in nebula::raftex::RaftexService::appendLog()+96 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RaftexService.cpp:200
#5  0x000000000496021e in nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog()+559 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RaftexService.cpp:131
#6  0x00000000049692b2 in nebula::raftex::cpp2::RaftexServiceAsyncProcessor::process_appendLog<apache::thrift::CompactProtocolReader, apache::thrift::CompactProtocolWriter>()+795 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RaftexService.tcc:115
#7  0x000000000498d1c4 in apache::thrift::RequestTask<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>::run()+315 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at AsyncProcessor.h:471
#8  0x00000000049684cb in apache::thrift::GeneratedAsyncProcessor::processInThread<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedCompressedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::raftex::cpp2::RaftexServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedCompressedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::raftex::cpp2::RaftexServiceAsyncProcessor*)::{lambda()#1}::operator()() const!()+56 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at AsyncProcessor.h:1114
#9  0x000000000497d324 in folly::detail::function::FunctionTraits<void()>::callSmall<apache::thrift::GeneratedAsyncProcessor::processInThread(apache::thrift::ResponseChannelRequest::UniquePtr, apache::thrift::SerializedCompressedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, apache::thrift::GeneratedAsyncProcessor::ProcessFunc<Derived>, ChildType*) [with ChildType = nebula::raftex::cpp2::RaftexServiceAsyncProcessor]::<lambda()> >()+35 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at Function.h:371
#10 0x0000000004e3d7ba in virtual thunk to apache::thrift::concurrency::FunctionRunner::run()!()+169 in /root/src/balance-fix-nebula/build/bin/nebula-storaged
#11 0x0000000004fb4259 in apache::thrift::concurrency::ThreadManager::Impl::Worker::run()!()+488 in /root/src/balance-fix-nebula/build/bin/nebula-storaged
#12 0x0000000004fb65c1 in apache::thrift::concurrency::PthreadThread::threadMain(void*)!()+224 in /root/src/balance-fix-nebula/build/bin/nebula-storaged
#13 0x00007f9149c84609 in start_thread()+216 in /lib/x86_64-linux-gnu/libpthread.so.0 at pthread_create.c:477
#14 0x00007f9149bab293 in __GI___clone!()+66 in /lib/x86_64-linux-gnu/libc.so.6 at clone.S:95
```

检查 RaftexService.cpp:165 附近的代码：

```cpp
std::shared_ptr<RaftPart> RaftexService::findPart(GraphSpaceID spaceId, PartitionID partId) {
  folly::RWSpinLock::ReadHolder rh(partsLock_);
  auto it = parts_.find(std::make_pair(spaceId, partId));
  if (it == parts_.end()) {
    // Part not found
    LOG_EVERY_N(WARNING, 100) << "Cannot find the part " << partId << " in the graph space "
                              << spaceId;
    return std::shared_ptr<RaftPart>();
  }
 
  // Otherwise, return the part pointer
  return it->second;
}
```

获取读锁阻塞了，
那应该是哪个地方拿了写锁没有释放？
列出所有使用 partsLock 的代码片段：

```cpp
void RaftexService::stop() {
  int expected = STATUS_RUNNING;
  if (!status_.compare_exchange_strong(expected, STATUS_NOT_RUNNING)) {
    return;
  }
 
  // stop service
  LOG(INFO) << "Stopping the raftex service on port " << serverPort_;
  {
    folly::RWSpinLock::WriteHolder wh(partsLock_);
    for (auto& p : parts_) {
      p.second->stop();
    }
    parts_.clear();
    LOG(INFO) << "All partitions have stopped";
  }
  server_->stop();
}
```

```cpp
void RaftexService::addPartition(std::shared_ptr<RaftPart> part) {
  // todo(doodle): If we need to start both listener and normal replica on same
  // hosts, this class need to be aware of type.
  folly::RWSpinLock::WriteHolder wh(partsLock_);
  parts_.emplace(std::make_pair(part->spaceId(), part->partitionId()), part);
}
 
void RaftexService::removePartition(std::shared_ptr<RaftPart> part) {
  folly::RWSpinLock::WriteHolder wh(partsLock_);
  parts_.erase(std::make_pair(part->spaceId(), part->partitionId()));
  // Stop the partition
  part->stop();
}
 
std::shared_ptr<RaftPart> RaftexService::findPart(GraphSpaceID spaceId, PartitionID partId) {
  folly::RWSpinLock::ReadHolder rh(partsLock_);
  auto it = parts_.find(std::make_pair(spaceId, partId));
  if (it == parts_.end()) {
    // Part not found
    LOG_EVERY_N(WARNING, 100) << "Cannot find the part " << partId << " in the graph space "
                              << spaceId;
    return std::shared_ptr<RaftPart>();
  }
 
  // Otherwise, return the part pointer
  return it->second;
}
```

合理猜测 void RaftexService::stop() 或者 void RaftexService::removePartition(std::shared_ptr<RaftPart> part) 中的 part→stop() 调用卡住了，
pstack 记录了 removePartition() 的一个调用堆栈：

```txt

thread: 0, lwp: 23, type: 0
#0  0x00007f9149c8b376 in __pthread_cond_wait()+534 in /lib/x86_64-linux-gnu/libpthread.so.0 at futex-internal.h:183
#1  0x00000000057d8f60 in std::condition_variable::wait(std::unique_lock<std::mutex>&)!()+15 in /root/src/balance-fix-nebula/build/bin/nebula-storaged
#2  0x000000000488d056 in std::condition_variable::wait<nebula::raftex::Host::waitForStop()::<lambda()> >()+59 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at condition_variable:101
#3  0x0000000004885679 in nebula::raftex::Host::waitForStop()+226 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at Host.cpp:42
#4  0x000000000482a963 in nebula::raftex::RaftPart::stop()+1380 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RaftPart.cpp:347
#5  0x0000000004881966 in nebula::raftex::RaftexService::removePartition()+177 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RaftexService.cpp:161
#6  0x00000000047d0470 in nebula::kvstore::NebulaStore::removePart()+379 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at NebulaStore.cpp:438
#7  0x00000000032e0214 in nebula::storage::RemovePartProcessor::process()+323 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at AdminProcessor.h:170
#8  0x00000000032e2fdf in nebula::storage::StorageAdminServiceHandler::future_removePart()+100 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at StorageAdminServiceHandler.cpp:50
#9  0x0000000003b8f9a0 in nebula::storage::cpp2::StorageAdminServiceSvIf::async_tm_removePart()+267 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at StorageAdminService.cpp:240
#10 0x0000000003b9ebb5 in nebula::storage::cpp2::StorageAdminServiceAsyncProcessor::process_removePart<apache::thrift::CompactProtocolReader, apache::thrift::CompactProtocolWriter>()+716 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at StorageAdminService.tcc:251
#11 0x0000000003bcd6ae in apache::thrift::RequestTask<nebula::storage::cpp2::StorageAdminServiceAsyncProcessor>::run()+315 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at AsyncProcessor.h:471
#12 0x0000000003b9c8c9 in apache::thrift::GeneratedAsyncProcessor::processInThread<nebula::storage::cpp2::StorageAdminServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedCompressedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::storage::cpp2::StorageAdminServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedCompressedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::storage::cpp2::StorageAdminServiceAsyncProcessor*)::{lambda()#1}::operator()() const!()+56 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at AsyncProcessor.h:1114
#13 0x0000000003bbc478 in folly::detail::function::FunctionTraits<void()>::callSmall<apache::thrift::GeneratedAsyncProcessor::processInThread(apache::thrift::ResponseChannelRequest::UniquePtr, apache::thrift::SerializedCompressedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, apache::thrift::GeneratedAsyncProcessor::ProcessFunc<Derived>, ChildType*) [with ChildType = nebula::storage::cpp2::StorageAdminServiceAsyncProcessor]::<lambda()> >()+35 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at Function.h:371
#14 0x0000000004e3d7ba in virtual thunk to apache::thrift::concurrency::FunctionRunner::run()!()+169 in /root/src/balance-fix-nebula/build/bin/nebula-storaged
#15 0x0000000004fb4259 in apache::thrift::concurrency::ThreadManager::Impl::Worker::run()!()+488 in /root/src/balance-fix-nebula/build/bin/nebula-storaged
#16 0x0000000004fb65c1 in apache::thrift::concurrency::PthreadThread::threadMain(void*)!()+224 in /root/src/balance-fix-nebula/build/bin/nebula-storaged
#17 0x00007f9149c84609 in start_thread()+216 in /lib/x86_64-linux-gnu/libpthread.so.0 at pthread_create.c:477
#18 0x00007f9149bab293 in __GI___clone!()+66 in /lib/x86_64-linux-gnu/libc.so.6 at clone.S:95
```

storage 应该是卡在 Host.cpp:42 这行代码上了，拉出来看下：

```cpp
void Host::waitForStop() {
  std::unique_lock<std::mutex> g(lock_);
 
  CHECK(stopped_);
  noMoreRequestCV_.wait(g, [this] { return !requestOnGoing_; });
  LOG(INFO) << idStr_ << "The host has been stopped!";
}
```

好眼熟啊，之前定位 [Nebula Raft 死锁问题分析](https://coderatwork.cn/posts/raft-deadlock/) 这个问题的时候，
也是卡在这个 cv 上。复习一下当时 Raft 是怎么死锁的：

1. worker thread1 中有
    + 回调 c1 尝试获取 raftLock_
    + 回调 c2 设置 requestOnGoing_ = false，且 c1 先于 c2 被调用 
2. 回调 c2 设置 requestOnGoing_ = false，且 c1 先于 c2 被调用 

然后 thread1、thread2 就死锁了。
但是这个场景里面似乎并没有尝试获取 raftLock_ 的地方。
不过可以换个思路：
之前是因为 worker thread 拿着 raftLock_ 在等待 requestOnGoing_ == false 导致的死锁，
如果它拿着另一个锁比如 partsLock_ 在等待 requestOnGoing_ == false 是否也可能导致死锁？
仔细看以上的堆栈：

```txt
thread: 0, lwp: 23, type: 0
#0  0x00007f9149c8b376 in __pthread_cond_wait()+534 in /lib/x86_64-linux-gnu/libpthread.so.0 at futex-internal.h:183
#1  0x00000000057d8f60 in std::condition_variable::wait(std::unique_lock<std::mutex>&)!()+15 in /root/src/balance-fix-nebula/build/bin/nebula-storaged
#2  0x000000000488d056 in std::condition_variable::wait<nebula::raftex::Host::waitForStop()::<lambda()> >()+59 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at condition_variable:101
#3  0x0000000004885679 in nebula::raftex::Host::waitForStop()+226 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at Host.cpp:42
#4  0x000000000482a963 in nebula::raftex::RaftPart::stop()+1380 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RaftPart.cpp:347
#5  0x0000000004881966 in nebula::raftex::RaftexService::removePartition()+177 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at RaftexService.cpp:161
#6  0x00000000047d0470 in nebula::kvstore::NebulaStore::removePart()+379 in /root/src/balance-fix-nebula/build/bin/nebula-storaged at NebulaStore.cpp:438
```

从线程 23 的调用栈上可以看出，
它正好拿着 partsLock 这个 spinlock 在等待 requestOnGoing == false（RaftService.cpp:161）。
除了 partsLock，从调用栈上可以看到 NebulaStore.cpp:438 也拿着个锁：

```cpp
void NebulaStore::removePart(GraphSpaceID spaceId, PartitionID partId) {
  folly::RWSpinLock::WriteHolder wh(&lock_);
  auto spaceIt = this->spaces_.find(spaceId);
  if (spaceIt != this->spaces_.end()) {
    auto partIt = spaceIt->second->parts_.find(partId);
    if (partIt != spaceIt->second->parts_.end()) {
      auto* e = partIt->second->engine();
      CHECK_NOTNULL(e);
      raftService_->removePartition(partIt->second);
      diskMan_->removePartFromPath(spaceId, partId, e->getDataRoot());
      partIt->second->resetPart();
      spaceIt->second->parts_.erase(partId);
      e->removePart(partId);
    }
  }
  LOG(INFO) << "Space " << spaceId << ", part " << partId << " has been removed!";
}
```

如果有另外一条路径是需要拿到 partsLock 才能设置 requestOnGoing == fasle，
那就能找到一条导致死锁的路径了。
这里需要解释一下，partsLock 锁的获取和 requestOnGoing = false 的设置并不是处在一条调用链上，
而是被调度倒同一个 worker thread 上的两个不同回调，
且获取锁的回调先于 requestOnGoing = false 这个回调被执行。
问题是，具体哪个 worker thread？
从 pstack 上并不容易看出，没办法，
上万能的 printf 调试大法。
requestOnGoing = false 这条语句本质上是在 Host.cpp 中的 appendLogsInternal()中执行的，
我们就在 appendLogsInternal() 中把回调运行的线程 pid 打印出来：

```cpp
void Host::appendLogsInternal(folly::EventBase* eb, std::shared_ptr<cpp2::AppendLogRequest> req) {
  using TransportException = apache::thrift::transport::TTransportException;
 
  // long long microseconds =
  // std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count();
  auto reqId = std::chrono::high_resolution_clock::now().time_since_epoch().count();
  pid_t thisTid = syscall(__NR_gettid);
  // LOG(INFO) << folly::format("append with req: {}, started within thread {}", reqId, thisTid);
  std::cerr << folly::format("append with req: {}, started within thread {}", reqId, thisTid)
            << std::endl;
  // LOG(INFO) << "append with req: " << reqId << " start within thread: " << tid;
  eb->runImmediatelyOrRunInEventBaseThreadAndWait([reqId]() {
    pid_t tid = syscall(__NR_gettid);
    // LOG(INFO) << folly::format("append log req {} will run within thread {}", reqId, tid);
    std::cerr << folly::format("append log req {} will run within thread {}", reqId, tid)
              << std::endl;
    // LOG(INFO) << "append log req " << reqId << " will run within thread " << tid;
  });
 
  sendAppendLogRequest(eb, req)
      .via(eb)
      .thenValue([eb, self = shared_from_this()](cpp2::AppendLogResponse&& resp) {
        pid_t tid = syscall(__NR_gettid);
        // LOG(INFO) << folly::format("append log req {} done within thread {}", reqId, tid);
        std::cerr << folly::format("append log req {} done within thread {}", reqId, tid)
                  << std::endl;
        // LOG(INFO) << "append log req " << reqId << " will done within thread " <<
        // std::this_thread::get_id();
 
        LOG_IF(INFO, FLAGS_trace_raft)
            << self->idStr_ << "AppendLogResponse "
            << "code " << apache::thrift::util::enumNameSafe(resp.get_error_code()) << ", currTerm "
            << resp.get_current_term() << ", lastLogTerm " << resp.get_last_matched_log_term()
            << ", commitLogId " << resp.get_committed_log_id() << ", lastLogIdSent_ "
            << self->lastLogIdSent_ << ", lastLogTermSent_ " << self->lastLogTermSent_;
        switch (resp.get_error_code()) {
          case cpp2::ErrorCode::SUCCEEDED:
          case cpp2::ErrorCode::E_LOG_GAP:
          case cpp2::ErrorCode::E_LOG_STALE: {
            VLOG(2) << self->idStr_ << "AppendLog request sent successfully";
 
            std::shared_ptr<cpp2::AppendLogRequest> newReq;
            {
              std::lock_guard<std::mutex> g(self->lock_);
              auto res = self->canAppendLog();
              if (res != cpp2::ErrorCode::SUCCEEDED) {
                cpp2::AppendLogResponse r;
                r.error_code_ref() = res;
                self->setResponse(r);
                return;
              }
              // Host is working
              self->lastLogIdSent_ = resp.get_last_matched_log_id();
              self->lastLogTermSent_ = resp.get_last_matched_log_term();
              self->followerCommittedLogId_ = resp.get_committed_log_id();
              if (self->lastLogIdSent_ < self->logIdToSend_) {
                // More to send
                VLOG(2) << self->idStr_ << "There are more logs to send";
                auto result = self->prepareAppendLogRequest();
                if (ok(result)) {
                  newReq = std::move(value(result));
                } else {
                  cpp2::AppendLogResponse r;
                  r.error_code_ref() = error(result);
                  self->setResponse(r);
                  return;
                }
              } else {
                // resp.get_last_matched_log_id() >= self->logIdToSend_
                // All logs up to logIdToSend_ has been sent, fulfill the promise
                self->promise_.setValue(resp);
                // Check if there are any pending request:
                // Eithor send pending requst if any, or set Host to vacant
                newReq = self->getPendingReqIfAny(self);
              }
            }
            if (newReq) {
              self->appendLogsInternal(eb, newReq);
            }
            return;
          }
          // Usually the peer is not in proper state, for example:
          // E_UNKNOWN_PART/E_BAD_STATE/E_NOT_READY/E_WAITING_SNAPSHOT
          // In this case, nothing changed, just return the error
          default: {
            LOG_EVERY_N(ERROR, 100)
                << self->idStr_ << "Failed to append logs to the host (Err: "
                << apache::thrift::util::enumNameSafe(resp.get_error_code()) << ")";
            {
              std::lock_guard<std::mutex> g(self->lock_);
              self->setResponse(resp);
            }
            return;
          }
        }
      })
      .thenError(
          folly::tag_t<TransportException>{},
          [self = shared_from_this(), req](TransportException&& ex) {
            pid_t tid = syscall(__NR_gettid);
            std::cerr << folly::format("append log req {} encounter exception {} within thread {}",
                                       reqId,
                                       ex.what(),
                                       tid)
                      << std::endl;
            //  LOG(INFO) << folly::format("append log req {} encounter exception {} within thread
            //  {}", reqId, ex.what(), tid); LOG(INFO) << "append log req " << reqId << " encounter
            //  exception:" <<  ex.what() << " within thread " << std::this_thread::get_id();
 
            VLOG(2) << self->idStr_ << ex.what();
            cpp2::AppendLogResponse r;
            r.error_code_ref() = cpp2::ErrorCode::E_RPC_EXCEPTION;
            {
              std::lock_guard<std::mutex> g(self->lock_);
              if (ex.getType() == TransportException::TIMED_OUT) {
                LOG_IF(INFO, FLAGS_trace_raft)
                    << self->idStr_ << "append log time out"
                    << ", space " << req->get_space() << ", part " << req->get_part()
                    << ", current term " << req->get_current_term() << ", committed_id "
                    << req->get_committed_log_id() << ", last_log_term_sent "
                    << req->get_last_log_term_sent() << ", last_log_id_sent "
                    << req->get_last_log_id_sent() << ", set lastLogIdSent_ to logIdToSend_ "
                    << self->logIdToSend_ << ", logs size " << req->get_log_str_list().size();
              }
              self->setResponse(r);
            }
            // a new raft log or heartbeat will trigger another appendLogs in Host
            return;
          })
      .thenError(folly::tag_t<std::exception>{}, [self = shared_from_this()](std::exception&& ex) {
        pid_t tid = syscall(__NR_gettid);
        // LOG(INFO) << folly::format("append log req {} encounter exception {} within thread {}",
        // reqId, ex.what(), tid);
        std::cerr << folly::format("append log req {} encounter exception {} within thread {}",
                                   reqId,
                                   ex.what(),
                                   tid)
                  << std::endl;
        // LOG(INFO) << "append log req " << reqId << " encounter exception:" <<  ex.what() << "
        // within thread " << std::this_thread::get_id();
 
        VLOG(2) << self->idStr_ << ex.what();
        cpp2::AppendLogResponse r;
        r.error_code_ref() = cpp2::ErrorCode::E_RPC_EXCEPTION;
        {
          std::lock_guard<std::mutex> g(self->lock_);
          self->setResponse(r);
        }
        // a new raft log or heartbeat will trigger another appendLogs in Host
        return;
      });
}
```

run again, storage 很快又扑街了：

```txt

(root@nebula) [ttos_3p3r]> show hosts
+----------+------+-----------+--------------+----------------------+------------------------+
| Host     | Port | Status    | Leader count | Leader distribution  | Partition distribution |
+----------+------+-----------+--------------+----------------------+------------------------+
| "store1" | 9779 | "OFFLINE" | 0            | "No valid partition" | "ttos_3p3r:446"        |
| "store2" | 9779 | "OFFLINE" | 0            | "No valid partition" | "ttos_3p3r:443"        |
| "store3" | 9779 | "ONLINE"  | 247          | "ttos_3p3r:247"      | "ttos_3p3r:432"        |
| "store4" | 9779 | "ONLINE"  | 108          | "ttos_3p3r:108"      | "ttos_3p3r:109"        |
| "store5" | 9779 | "ONLINE"  | 20           | "ttos_3p3r:20"       | "ttos_3p3r:106"        |
| "Total"  |      |           | 375          | "ttos_3p3r:375"      | "ttos_3p3r:1536"       |
+----------+------+-----------+--------------+----------------------+------------------------+
Got 6 rows (time spent 9484/10020 us)
 
Tue, 11 Jan 2022 12:21:45 CST
 
(root@nebula) [ttos_3p3r]>
```

查看 store1 的日志：

```txt
# tail storaged-stderr.log
append log req 1641874628611367114 will run within thread 71
append log req 1641874628611367114 done within thread 71
append with req: 1641874628614040712, started within thread 71
append log req 1641874628614040712 will run within thread 71
append log req 1641874628614040712 done within thread 71
append with req: 1641874628616965551, started within thread 71
append log req 1641874628616965551 will run within thread 71
append log req 1641874628616965551 done within thread 71
append with req: 1641874628619785706, started within thread 71
append log req 1641874628619785706 will run within thread 71
```

应该是卡在线程 71 上了，gdb attach 上去看这家伙在干嘛，呵呵呵，果然不出所料，partsLock_ 搞出一个死循环了：

```txt

(gdb) bt
#0  0x00007f8ad6e2289b in sched_yield () at ../sysdeps/unix/syscall-template.S:78
#1  0x00000000030eeb00 in __gthread_yield () at /usr/include/x86_64-linux-gnu/c++/9/bits/gthr-default.h:693
#2  0x00000000030ef6ce in std::this_thread::yield () at /usr/include/c++/9/thread:356
#3  0x0000000003530c69 in folly::RWSpinLock::lock_shared (this=0x75fdf48) at /data/src/raft-nebula/build/third-party/install/include/folly/synchronization/RWSpinLock.h:212
#4  0x0000000003530e65 in folly::RWSpinLock::ReadHolder::ReadHolder (this=0x7f89fe7f2780, lock=...) at /data/src/raft-nebula/build/third-party/install/include/folly/synchronization/RWSpinLock.h:320
#5  0x00000000048879fd in nebula::raftex::RaftexService::findPart (this=0x75fdee0, spaceId=1, partId=352) at /data/src/balance-fix-nebula/src/kvstore/raftex/RaftexService.cpp:165
#6  0x00000000048883ce in nebula::raftex::RaftexService::async_eb_heartbeat (this=0x75fdee0, callback=..., req=...) at /data/src/balance-fix-nebula/src/kvstore/raftex/RaftexService.cpp:226
#7  0x0000000004977a32 in nebula::raftex::cpp2::RaftexServiceAsyncProcessor::process_heartbeat<apache::thrift::CompactProtocolReader, apache::thrift::CompactProtocolWriter> (this=0x7f89e4032aa0, req=..., serializedRequest=..., ctx=0x7f89e4057e08, eb=0x7f89e4000e30, tm=0x756e250)
    at /data/src/balance-fix-nebula/build/src/interface/gen-cpp2/RaftexService.tcc:229
#8  0x00000000049716c3 in nebula::raftex::cpp2::RaftexServiceAsyncProcessor::setUpAndProcess_heartbeat<apache::thrift::CompactProtocolReader, apache::thrift::CompactProtocolWriter> (this=0x7f89e4032aa0, req=..., serializedRequest=..., ctx=0x7f89e4057e08, eb=0x7f89e4000e30,
    tm=0x756e250) at /data/src/balance-fix-nebula/build/src/interface/gen-cpp2/RaftexService.tcc:203
#9  0x0000000004975101 in apache::thrift::detail::ap::nonRecursiveProcess<apache::thrift::CompactProtocolReader, nebula::raftex::cpp2::RaftexServiceAsyncProcessor> (processor=0x7f89e4032aa0, req=..., serializedRequest=..., untypedMethodMetadata=..., ctx=0x7f89e4057e08,
    eb=0x7f89e4000e30, tm=0x756e250) at /data/src/raft-nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:922
#10 0x0000000004970a3c in apache::thrift::detail::ap::process<nebula::raftex::cpp2::RaftexServiceAsyncProcessor> (processor=0x7f89e4032aa0, req=..., serializedRequest=..., methodMetadata=..., protType=apache::thrift::protocol::T_COMPACT_PROTOCOL, ctx=0x7f89e4057e08,
    eb=0x7f89e4000e30, tm=0x756e250) at /data/src/raft-nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:949
#11 0x000000000496e2af in nebula::raftex::cpp2::RaftexServiceAsyncProcessor::processSerializedCompressedRequestWithMetadata (this=0x7f89e4032aa0, req=..., serializedRequest=..., methodMetadata=..., protType=apache::thrift::protocol::T_COMPACT_PROTOCOL, context=0x7f89e4057e08,
    eb=0x7f89e4000e30, tm=0x756e250) at /data/src/balance-fix-nebula/build/src/interface/gen-cpp2/RaftexService.cpp:294
#12 0x0000000004f34126 in ?? ()
#13 0x0000000004f111cd in ?? ()
#14 0x0000000004f11efa in apache::thrift::rocket::ThriftRocketServerHandler::handleRequestResponseFrame(apache::thrift::rocket::RequestResponseFrame&&, apache::thrift::rocket::RocketServerFrameContext&&) ()
#15 0x0000000004efeb2f in apache::thrift::rocket::RocketServerConnection::handleFrame(std::unique_ptr<folly::IOBuf, std::default_delete<folly::IOBuf> >) ()
#16 0x0000000004f07046 in apache::thrift::rocket::Parser<apache::thrift::rocket::RocketServerConnection>::readDataAvailableOld(unsigned long) ()
#17 0x0000000004f076e9 in apache::thrift::rocket::Parser<apache::thrift::rocket::RocketServerConnection>::readDataAvailable(unsigned long) ()
#18 0x00000000052a8f8b in folly::AsyncSocket::handleRead() ()
#19 0x000000000529da0e in folly::AsyncSocket::ioReady(unsigned short) ()
#20 0x000000000537a653 in ?? ()
#21 0x000000000537ad77 in event_base_loop ()
#22 0x00000000052b88b8 in folly::EventBase::loopBody(int, bool) ()
#23 0x00000000052b94b2 in folly::EventBase::loop() ()
#24 0x00000000052bc1cc in folly::EventBase::loopForever() ()
#25 0x00000000052404a1 in folly::IOThreadPoolExecutor::threadRun(std::shared_ptr<folly::ThreadPoolExecutor::Thread>) ()
#26 0x000000000524f6cb in void folly::detail::function::FunctionTraits<void ()>::callBig<std::_Bind<void (folly::ThreadPoolExecutor::*(folly::ThreadPoolExecutor*, std::shared_ptr<folly::ThreadPoolExecutor::Thread>))(std::shared_ptr<folly::ThreadPoolExecutor::Thread>)> >(folly::detail::function::Data&) ()
#27 0x00000000030d1640 in folly::detail::function::FunctionTraits<void ()>::operator()() (this=0x75bc720) at /data/src/raft-nebula/build/third-party/install/include/folly/Function.h:400
#28 0x00000000030f06b0 in folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}::operator()() (__closure=0x75bc720) at /data/src/raft-nebula/build/third-party/install/include/folly/executors/thread_factory/NamedThreadFactory.h:40
#29 0x0000000003151961 in std::__invoke_impl<void, folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}>(std::__invoke_other, folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}&&) (__f=...) at /usr/include/c++/9/bits/invoke.h:60
#30 0x0000000003150870 in std::__invoke<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}>(std::__invoke_result&&, (folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}&&)...) (__fn=...) at /usr/include/c++/9/bits/invoke.h:95
#31 0x000000000314f980 in std::thread::_Invoker<std::tuple<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}> >::_M_invoke<0ul>(std::_Index_tuple<0ul>) (this=0x75bc720) at /usr/include/c++/9/thread:244
#32 0x000000000314ecf0 in std::thread::_Invoker<std::tuple<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}> >::operator()() (this=0x75bc720) at /usr/include/c++/9/thread:251
#33 0x000000000314ba6c in std::thread::_State_impl<std::thread::_Invoker<std::tuple<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}> > >::_M_run() (this=0x75bc710) at /usr/include/c++/9/thread:195
#34 0x000000000585df34 in execute_native_thread_routine ()
#35 0x00007f8ad6f18609 in start_thread (arg=<optimized out>) at pthread_create.c:477
#36 0x00007f8ad6e3f293 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
(gdb)
```

再详细解释一下为什么这里会死锁：

1. storage 收到一个 heartbeat 请求，这个请求被调度倒 eb 所在的线程去执行
2. heartbeat 调用 findPart() 确定对应的分区，findPart() 尝试获取 partsLock_ 这个 spinlock
3. heartbeat 执行结束前，storage 上的一个 appendLogInternal 请求也被调度倒同样的 eb 上执行，appendLogInternal() 执行中会设置 requestOnGoing_ = false
4. 另一边一个 removePartition() 请求正在执行中，他已经拿到 partsLock_ 等待 part.stop()，part.stop() 又等待 requestOnGoing_ = false

这样，一个死锁链条就完美诞生了：

process_heartbeat()→ [等待释放 partsLock_]→ part.stop()→ [等待设置 requestOnGoing_ = false]→ appendLogsInternal()→ [等待 eb 结束执行 process_heartbeat()]
