---
title: "Nebula Raft 死锁问题分析"
date: 2021-11-17T21:59:08+08:00
draft: false
---

最近几周在测试 Nebula 的时候经常碰到 raft storage 节点莫名离线的问题。
具体情况是这样的: 起一个 5storage 节点的 raft 集群，
然后开插边压测程序，同时不停的制造 leader change 的场景，
通常十分钟以内就能看到至少一个 storage 节点变成离线状态，
但是节点对应的进程却还活着的：

```txt
(root@nebula) [(none)]> show hosts;
+-----------------+-------+-----------+--------------+----------------------+------------------------+
| Host            | Port  | Status    | Leader count | Leader distribution  | Partition distribution |
+-----------------+-------+-----------+--------------+----------------------+------------------------+
| "192.168.15.11" | 33299 | "OFFLINE" | 0            | "No valid partition" | "ttos_3p3r:1"          |
+-----------------+-------+-----------+--------------+----------------------+------------------------+
| "192.168.15.11" | 54889 | "ONLINE"  | 0            | "No valid partition" | "ttos_3p3r:1"          |
+-----------------+-------+-----------+--------------+----------------------+------------------------+
| "192.168.15.11" | 34679 | "ONLINE"  | 1            | "ttos_3p3r:1"        | "ttos_3p3r:1"          |
+-----------------+-------+-----------+--------------+----------------------+------------------------+
| "192.168.15.11" | 57211 | "ONLINE"  | 0            | "No valid partition" | "ttos_3p3r:1"          |
+-----------------+-------+-----------+--------------+----------------------+------------------------+
| "192.168.15.11" | 35767 | "ONLINE"  | 0            | "No valid partition" | "ttos_3p3r:1"          |
+-----------------+-------+-----------+--------------+----------------------+------------------------+
| "Total"         |       |           | 1            | "ttos_3p3r:1"        | "ttos_3p3r:5"          |
+-----------------+-------+-----------+--------------+----------------------+------------------------+
Got 6 rows (time spent 1094/12349 us)
 
Wed, 03 Nov 2021 11:23:48 CST
```

```txt
# ps aux | grep 33299 | grep -v grep
root     2470607  184  0.0 1385496 159800 ?      Ssl  10:55  59:11 /data/src/wwl/nebula/build/bin/nebula-storaged --flagfile /data/src/wwl/test/etc/nebula-storaged.conf --pid_file /data/src/wwl/test/pids/nebula-storaged.pid.4 --meta_server_addrs 192.168.15.11:9559 --heartbeat_interval_secs 1 --raft_heartbeat_interval_secs 1 --minloglevel 3 --log_dir /data/src/wwl/test/logs/storaged.4 --local_ip 192.168.15.11 --port 33299 --ws_http_port 53553 --ws_h2_port 46147 --data_path /data/src/wwl/test/data/storaged.4
```

通过 pstack 查看进程的堆栈观察到这个 storage 实例中的某个线程 2470643 卡在一个条件变量上一直不动了：

```txt
Thread 37 (Thread 0x7fc8d23fd700 (LWP 2470643) "executor-pri3-3"):
...
#11 0x00007fc8e0f159fd in clone () from /lib64/libc.so.6
Thread 36 (Thread 0x7fc8d24fe700 (LWP 2470642) "executor-pri3-2"):
#0  0x00007fc8e11f0a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
#1  0x0000000004ba7a3c in std::condition_variable::wait(std::unique_lock<std::mutex>&) ()
#2  0x0000000003da583e in std::condition_variable::wait<nebula::raftex::Host::reset()::{lambda()#1}>(std::unique_lock<std::mutex>&, nebula::raftex::Host::reset()::{lambda()#1}) (this=0x7fc8c543d3b0, __lock=..., __p=...) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/condition_variable:99
#3  0x0000000003d91965 in nebula::raftex::Host::reset (this=0x7fc8c543d310) at /root/nebula-workspace/nebula/src/kvstore/raftex/Host.h:44
#4  0x0000000003d9da15 in nebula::raftex::RaftPart::handleElectionResponses (this=0x7fc8c54df010, voteReq=..., resps=..., hosts=..., proposedTerm=45) at /root/nebula-workspace/nebula/src/kvstore/raftex/RaftPart.cpp:1145
#5  0x0000000003d9cde0 in nebula::raftex::RaftPart::<lambda(auto:132&&)>::operator()<folly::Try<std::vector<std::pair<long unsigned int, nebula::raftex::cpp2::AskForVoteResponse> > > >(folly::Try<std::vector<std::pair<unsigned long, nebula::raftex::cpp2::AskForVoteResponse>, std::allocator<std::pair<unsigned long, nebula::raftex::cpp2::AskForVoteResponse> > > > &&) (__closure=0x7fc8c4c11320, t=...) at /root/nebula-workspace/nebula/src/kvstore/raftex/RaftPart.cpp:1123
#6  0x0000000003db1421 in folly::Future<std::vector<std::pair<unsigned long, nebula::raftex::cpp2::AskForVoteResponse>, std::allocator<std::pair<unsigned long, nebula::raftex::cpp2::AskForVoteResponse> > > >::<lambda(folly::Executor::KeepAlive<folly::Executor>&&, folly::Try<std::vector<std::pair<long unsigned int, nebula::raftex::cpp2::AskForVoteResponse>, std::allocator<std::pair<long unsigned int, nebula::raftex::cpp2::AskForVoteResponse> > > >&&)>::operator()(folly::Executor::KeepAlive<folly::Executor> &&, folly::Try<std::vector<std::pair<unsigned long, nebula::raftex::cpp2::AskForVoteResponse>, std::allocator<std::pair<unsigned long, nebula::raftex::cpp2::AskForVoteResponse> > > > &&) (__closure=0x7fc8c4c11320, t=...) at /data/src/wwl/nebula/build/third-party/install/include/folly/futures/Future-inl.h:947
```

我们看 src/kvstore/raftex/Host.h:44 的具体代码，通过分析我们可以知道这个函数正在等待当前所有的 append log 请求结束，也就是 44 行对应的 noMoreRequestCV_.wait() 调用，它一直在等待 requestOnGoing_ 变为 false：

```c++
void reset() {
  std::unique_lock<std::mutex> g(lock_);
  noMoreRequestCV_.wait(g, [this] { return !requestOnGoing_; });
  logIdToSend_ = 0;
  logTermToSend_ = 0;
  lastLogIdSent_ = 0;
  lastLogTermSent_ = 0;
  committedLogId_ = 0;
  sendingSnapshot_ = false;
  followerCommittedLogId_ = 0;
}
```

如果我们继续看堆栈上的前一个调用，可以发现 Host.reset() 调用前，
RaftPart::handleElectionResponses() 在 1141 这行代码获取了 raftLock_ 这个锁，
我们看 src/kvstore/raftex/RaftPart.cpp:1145 中的具体代码：

```c++
bool RaftPart::handleElectionResponses(const cpp2::AskForVoteRequest& voteReq,
                                       const ElectionResponses& resps,
                                       const std::vector<std::shared_ptr<Host>>& hosts,
                                       TermID proposedTerm) {
  // Process the responses
  switch (processElectionResponses(resps, std::move(hosts), proposedTerm)) {
    case Role::LEADER: {
      // Elected
      LOG(INFO) << idStr_ << "The partition is elected as the leader";
      {
        std::lock_guard<std::mutex> g(raftLock_);
        if (status_ == Status::RUNNING) {
          leader_ = addr_;
          for (auto& host : hosts_) {
            host->reset();
          }
          bgWorkers_->addTask(
              [self = shared_from_this(), term = voteReq.get_term()] { self->onElected(term); });
          lastMsgAcceptedTime_ = 0;
        }
        weight_ = 1;
        commitInThisTerm_ = false;
      }
      sendHeartbeat();
      return true;
    }
...
```

进程不动，说明 requestOnGoing_ 一直都是 true 状态，通过 gdb attach 进去我们验证了这个猜测：

```txt
0x00007fc8e11f0a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
(gdb) up
#1  0x0000000004ba7a3c in std::condition_variable::wait(std::unique_lock<std::mutex>&) ()
(gdb) up
#2  0x0000000003da583e in std::condition_variable::wait<nebula::raftex::Host::reset()::{lambda()#1}>(std::unique_lock<std::mutex>&, nebula::raftex::Host::reset()::{lambda()#1}) (this=0x7fc8c543d3b0, __lock=..., __p=...) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/condition_variable:99
99        wait(__lock);
(gdb) up
#3  0x0000000003d91965 in nebula::raftex::Host::reset (this=0x7fc8c543d310) at /root/nebula-workspace/nebula/src/kvstore/raftex/Host.h:44
44      noMoreRequestCV_.wait(g, [this] { return !requestOnGoing_; });
(gdb) p requestOnGoing_
$1 = true
(gdb)
```

为什么 requestOnGoing_ 一直都是 true 状态呢？
通过翻阅 src/kvstore/raftex/Host.cpp 中的代码我们可以发现当存在 append log 请求时 requestOnGoing_ 在 Host::appendLogs() 函数中会被设置为 true，
当 append log 请求都结束时，这个变量在 Host::appendLogsInternal() 函数中会被设置为 fasle。
requestOnGoing_ 值一直不变，那么一个合理的猜测是某个 append log 请求卡住了。
我们重点关注 Host::appendLogsInternal() 这个函数的实现细节：

```c++
void Host::appendLogsInternal(folly::EventBase* eb, std::shared_ptr<cpp2::AppendLogRequest> req) {
  using TransportException = apache::thrift::transport::TTransportException;
  // long long microseconds = std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count();
  auto reqId = std::chrono::high_resolution_clock::now().time_since_epoch().count();
  pid_t thisTid = syscall(__NR_gettid);
  // LOG(INFO) << folly::format("append with req: {}, started within thread {}", reqId, thisTid);
  std::cerr << folly::format("append with req: {}, started within thread {}", reqId, thisTid) << std::endl;
  // LOG(INFO) << "append with req: " << reqId << " start within thread: " << tid;
  eb->runImmediatelyOrRunInEventBaseThreadAndWait([reqId]() {
    pid_t tid = syscall(__NR_gettid);
    // LOG(INFO) << folly::format("append log req {} will run within thread {}", reqId, tid);
    std::cerr << folly::format("append log req {} will run within thread {}", reqId, tid) << std::endl;
    // LOG(INFO) << "append log req " << reqId << " will run within thread " << tid;
  });
 
  sendAppendLogRequest(eb, req)
      .via(eb)
      .thenValue([eb, self = shared_from_this(), reqId](cpp2::AppendLogResponse&& resp) {
        pid_t tid = syscall(__NR_gettid);
        // LOG(INFO) << folly::format("append log req {} done within thread {}", reqId, tid);
        std::cerr << folly::format("append log req {} done within thread {}", reqId, tid) << std::endl;
        // LOG(INFO) << "append log req " << reqId << " will done within thread " << std::this_thread::get_id();
        LOG_IF(INFO, FLAGS_trace_raft)
            << self->idStr_ << "AppendLogResponse "
            << "code " << apache::thrift::util::enumNameSafe(resp.get_error_code()) << ", currTerm "
            << resp.get_current_term() << ", lastLogId " << resp.get_last_log_id()
            << ", lastLogTerm " << resp.get_last_log_term() << ", commitLogId "
            << resp.get_committed_log_id() << ", lastLogIdSent_ " << self->lastLogIdSent_
            << ", lastLogTermSent_ " << self->lastLogTermSent_;
        switch (resp.get_error_code()) {
          case cpp2::ErrorCode::SUCCEEDED:
          case cpp2::ErrorCode::E_LOG_GAP:
          case cpp2::ErrorCode::E_LOG_STALE: {
            VLOG(2) << self->idStr_ << "AppendLog request sent successfully";
 
            std::shared_ptr<cpp2::AppendLogRequest> newReq;
            {
              std::lock_guard<std::mutex> g(self->lock_);
              auto res = self->checkStatus();
              if (res != cpp2::ErrorCode::SUCCEEDED) {
                cpp2::AppendLogResponse r;
                r.set_error_code(res);
                self->setResponse(r);
                return;
              }
              // Host is working
              self->lastLogIdSent_ = resp.get_last_log_id();
              self->lastLogTermSent_ = resp.get_last_log_term();
              self->followerCommittedLogId_ = resp.get_committed_log_id();
              if (self->lastLogIdSent_ < self->logIdToSend_) {
                // More to send
                VLOG(2) << self->idStr_ << "There are more logs to send";
                auto result = self->prepareAppendLogRequest();
                if (ok(result)) {
                  newReq = std::move(value(result));
                } else {
                  cpp2::AppendLogResponse r;
                  r.set_error_code(error(result));
                  self->setResponse(r);
                  return;
                }
              } else {
                // resp.get_last_log_id() >= self->logIdToSend_
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
      .thenError(folly::tag_t<TransportException>{},
                 [reqId, self = shared_from_this(), req](TransportException&& ex) {
                   pid_t tid = syscall(__NR_gettid);
                   std::cerr << folly::format("append log req {} encounter exception {} within thread {}", reqId, ex.what(), tid) << std::endl;
                  //  LOG(INFO) << folly::format("append log req {} encounter exception {} within thread {}", reqId, ex.what(), tid);
                   //  LOG(INFO) << "append log req " << reqId << " encounter exception:" <<  ex.what() << " within thread " << std::this_thread::get_id();
                   VLOG(2) << self->idStr_ << ex.what();
                   cpp2::AppendLogResponse r;
                   r.set_error_code(cpp2::ErrorCode::E_RPC_EXCEPTION);
                   {
                     std::lock_guard<std::mutex> g(self->lock_);
                     if (ex.getType() == TransportException::TIMED_OUT) {
                       LOG_IF(INFO, FLAGS_trace_raft)
                           << self->idStr_ << "append log time out"
                           << ", space " << req->get_space() << ", part " << req->get_part()
                           << ", current term " << req->get_current_term() << ", last_log_id "
                           << req->get_last_log_id() << ", committed_id "
                           << req->get_committed_log_id() << ", last_log_term_sent "
                           << req->get_last_log_term_sent() << ", last_log_id_sent "
                           << req->get_last_log_id_sent() << ", set lastLogIdSent_ to logIdToSend_ "
                           << self->logIdToSend_ << ", logs size "
                           << req->get_log_str_list().size();
                     }
                     self->setResponse(r);
                   }
                   // a new raft log or heartbeat will trigger another appendLogs in Host
                   return;
                 })
      .thenError(folly::tag_t<std::exception>{}, [self = shared_from_this(), reqId](std::exception&& ex) {
        pid_t tid = syscall(__NR_gettid);
        // LOG(INFO) << folly::format("append log req {} encounter exception {} within thread {}", reqId, ex.what(), tid);
        std::cerr << folly::format("append log req {} encounter exception {} within thread {}", reqId, ex.what(), tid) << std::endl;
        // LOG(INFO) << "append log req " << reqId << " encounter exception:" <<  ex.what() << " within thread " << std::this_thread::get_id();
        VLOG(2) << self->idStr_ << ex.what();
        cpp2::AppendLogResponse r;
        r.set_error_code(cpp2::ErrorCode::E_RPC_EXCEPTION);
        {
          std::lock_guard<std::mutex> g(self->lock_);
          self->setResponse(r);
        }
        // a new raft log or heartbeat will trigger another appendLogs in Host
        return;
      });
}
 
ErrorOr<cpp2::ErrorCode, std::shared_ptr<cpp2::AppendLogRequest>> Host::prepareAppendLogRequest() {
  CHECK(!lock_.try_lock());
  VLOG(2) << idStr_ << "Prepare AppendLogs request from Log " << lastLogIdSent_ + 1 << " to "
          << logIdToSend_;
  if (lastLogIdSent_ + 1 > part_->wal()->lastLogId()) {
    LOG_IF(INFO, FLAGS_trace_raft)
        << idStr_ << "My lastLogId in wal is " << part_->wal()->lastLogId()
        << ", but you are seeking " << lastLogIdSent_ + 1
        << ", so i have nothing to send, logIdToSend_ = " << logIdToSend_;
    return cpp2::ErrorCode::E_NO_WAL_FOUND;
  }
  auto it = part_->wal()->iterator(lastLogIdSent_ + 1, logIdToSend_);
  if (it->valid()) {
    auto term = it->logTerm();
    auto req = std::make_shared<cpp2::AppendLogRequest>();
    req->set_space(part_->spaceId());
    req->set_part(part_->partitionId());
    req->set_current_term(logTermToSend_);
    req->set_last_log_id(logIdToSend_);
    req->set_leader_addr(part_->address().host);
    req->set_leader_port(part_->address().port);
    req->set_committed_log_id(committedLogId_);
    req->set_last_log_term_sent(lastLogTermSent_);
    req->set_last_log_id_sent(lastLogIdSent_);
    req->set_log_term(term);
 
    std::vector<cpp2::LogEntry> logs;
    for (size_t cnt = 0;
         it->valid() && it->logTerm() == term && cnt < FLAGS_max_appendlog_batch_size;
         ++(*it), ++cnt) {
      cpp2::LogEntry le;
      le.set_cluster(it->logSource());
      le.set_log_str(it->logMsg().toString());
      logs.emplace_back(std::move(le));
    }
    req->set_log_str_list(std::move(logs));
    return req;
  } else {
    if (!sendingSnapshot_) {
      LOG(INFO) << idStr_ << "Can't find log " << lastLogIdSent_ + 1 << " in wal, send the snapshot"
                << ", logIdToSend = " << logIdToSend_
                << ", firstLogId in wal = " << part_->wal()->firstLogId()
                << ", lastLogId in wal = " << part_->wal()->lastLogId();
      sendingSnapshot_ = true;
      part_->snapshot_->sendSnapshot(part_, addr_)
          .thenValue([self = shared_from_this()](auto&& status) {
            std::lock_guard<std::mutex> g(self->lock_);
            if (status.ok()) {
              auto commitLogIdAndTerm = status.value();
              self->lastLogIdSent_ = commitLogIdAndTerm.first;
              self->lastLogTermSent_ = commitLogIdAndTerm.second;
              self->followerCommittedLogId_ = commitLogIdAndTerm.first;
              LOG(INFO) << self->idStr_ << "Send snapshot succeeded!"
                        << " commitLogId = " << commitLogIdAndTerm.first
                        << " commitLogTerm = " << commitLogIdAndTerm.second;
            } else {
              LOG(INFO) << self->idStr_ << "Send snapshot failed!";
              // TODO(heng): we should tell the follower i am failed.
            }
            self->sendingSnapshot_ = false;
            self->noMoreRequestCV_.notify_all();
          });
    } else {
      LOG_EVERY_N(INFO, 100) << idStr_ << "The snapshot req is in queue, please wait for a moment";
    }
    return cpp2::ErrorCode::E_WAITING_SNAPSHOT;
  }
}
```

这个函数本质上干的活是：
1. 通过 sendAppendLogRequest() 向 raft peer 发起 append log rpc 请求
2. 回调处理 append log rpc 的结果，处理完了顺便在这里吧  requestOnGoing_ 变量设置为 false

卡住的一种可能是 rpc 回调一直没有返回，但是这边不大可能，
因为我们给 rpc 链接请求都设置了超时，所以这一点基本可以排除。
再观察这个函数，我们可以看到 sendAppendLogRequest(eb, req) 和它的回调处理用的都是在同一个 eb 中执行，
会不会是回调线程中的操作导致死锁了？代码翻了 N 遍，看不出明显的关联关系，只能通过打日志进一步观察运行细节。
我们在 Host::appendLogsInternal() 中把它运行的线程、eb 运行的线程以及回调运行的线程都打出来，
具体对应以上代码中的 167 行、181 行、254 行、282 行。
重新跑测试，很快我们又观察到死锁的情况，通过死锁进程的日志，
从打出的日志上我们看到 Host::appendLogsInternal() 确实卡住了：

```txt
...
append log req 1635908498110971639 done within thread 2470665
append with req: 1635908526021106910, started within thread 2470665
append log req 1635908526021106910 will run within thread 2470665
```

Host::appendLogsInternal() 和 eb 运行的线程居然是一样的，
都是 2470665，pstack 进去看 2470665 这个进程在干嘛：

```txt
Thread 1 (Thread 0x7fc8c15ff700 (LWP 2470665) "IOThreadPool9"):
#0  0x00007fc8e11f354d in __lll_lock_wait () from /lib64/libpthread.so.0
#1  0x00007fc8e11eee9b in _L_lock_883 () from /lib64/libpthread.so.0
#2  0x00007fc8e11eed68 in pthread_mutex_lock () from /lib64/libpthread.so.0
#3  0x0000000002a655d4 in __gthread_mutex_lock (__mutex=0x7fc8c54df150) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/x86_64-vesoft-linux/bits/gthr-default.h:748
#4  0x0000000002a658d6 in std::mutex::lock (this=0x7fc8c54df150) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/bits/std_mutex.h:103
#5  0x0000000002a6b43f in std::lock_guard<std::mutex>::lock_guard (this=0x7fc8c15fbbb8, __m=...) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/bits/std_mutex.h:162
#6  0x0000000003da1de2 in nebula::raftex::RaftPart::processHeartbeatRequest (this=0x7fc8c54df010, req=..., resp=...) at /root/nebula-workspace/nebula/src/kvstore/raftex/RaftPart.cpp:1650
#7  0x0000000003de1822 in nebula::raftex::RaftexService::async_eb_heartbeat (this=0x7fc8e0a32ab0, callback=..., req=...) at /root/nebula-workspace/nebula/src/kvstore/raftex/RaftexService.cpp:220
#8  0x0000000003e931dd in nebula::raftex::cpp2::RaftexServiceAsyncProcessor::process_heartbeat<apache::thrift::CompactProtocolReader, apache::thrift::CompactProtocolWriter> (this=0x7fc8d1702160, req=..., serializedRequest=..., ctx=0x7fc8c0940b10, eb=0x7fc8c0804000, tm=0x7fc8e0a142b0) at /root/nebula-workspace/nebula/build/src/interface/gen-cpp2/RaftexService.tcc:220
#9  0x0000000003e8ec96 in nebula::raftex::cpp2::RaftexServiceAsyncProcessor::setUpAndProcess_heartbeat<apache::thrift::CompactProtocolReader, apache::thrift::CompactProtocolWriter> (this=0x7fc8d1702160, req=..., serializedRequest=..., ctx=0x7fc8c0940b10, eb=0x7fc8c0804000, tm=0x7fc8e0a142b0) at /root/nebula-workspace/nebula/build/src/interface/gen-cpp2/RaftexService.tcc:198
#10 0x0000000003e90d57 in apache::thrift::detail::ap::process_pmap<apache::thrift::CompactProtocolReader, nebula::raftex::cpp2::RaftexServiceAsyncProcessor> (proc=0x7fc8d1702160, pmap=..., req=..., serializedRequest=..., ctx=0x7fc8c0940b10, eb=0x7fc8c0804000, tm=0x7fc8e0a142b0) at /data/src/wwl/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:891
#11 0x0000000003e8df5f in apache::thrift::detail::ap::process<nebula::raftex::cpp2::RaftexServiceAsyncProcessor> (processor=0x7fc8d1702160, req=..., serializedRequest=..., protType=apache::thrift::protocol::T_COMPACT_PROTOCOL, ctx=0x7fc8c0940b10, eb=0x7fc8c0804000, tm=0x7fc8e0a142b0) at /data/src/wwl/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:919
#12 0x0000000003e8cc0a in nebula::raftex::cpp2::RaftexServiceAsyncProcessor::processSerializedRequest (this=0x7fc8d1702160, req=..., serializedRequest=..., protType=apache::thrift::protocol::T_COMPACT_PROTOCOL, context=0x7fc8c0940b10, eb=0x7fc8c0804000, tm=0x7fc8e0a142b0) at /root/nebula-workspace/nebula/build/src/interface/gen-cpp2/RaftexService.cpp:102
#13 0x0000000004363be4 in apache::thrift::Cpp2Connection::requestReceived(std::unique_ptr<apache::thrift::HeaderServerChannel::HeaderRequest, std::default_delete<apache::thrift::HeaderServerChannel::HeaderRequest> >&&) ()
#14 0x000000000436f86c in apache::thrift::HeaderServerChannel::messageReceived(std::unique_ptr<folly::IOBuf, std::default_delete<folly::IOBuf> >&&, std::unique_ptr<apache::thrift::transport::THeader, std::default_delete<apache::thrift::transport::THeader> >&&) ()
#15 0x0000000004330c96 in apache::thrift::Cpp2Channel::read(wangle::HandlerContext<int, std::pair<std::unique_ptr<folly::IOBuf, std::default_delete<folly::IOBuf> >, apache::thrift::transport::THeader*> >*, std::pair<std::unique_ptr<folly::IOBuf, std::default_delete<folly::IOBuf> >, std::unique_ptr<apache::thrift::transport::THeader, std::default_delete<apache::thrift::transport::THeader> > >) ()
#16 0x000000000433e213 in wangle::ContextImpl<apache::thrift::Cpp2Channel>::read(std::pair<std::unique_ptr<folly::IOBuf, std::default_delete<folly::IOBuf> >, std::unique_ptr<apache::thrift::transport::THeader, std::default_delete<apache::thrift::transport::THeader> > >) ()
#17 0x000000000433e105 in wangle::ContextImpl<apache::thrift::FramingHandler>::fireRead(std::pair<std::unique_ptr<folly::IOBuf, std::default_delete<folly::IOBuf> >, std::unique_ptr<apache::thrift::transport::THeader, std::default_delete<apache::thrift::transport::THeader> > >) ()
#18 0x000000000433ed85 in apache::thrift::FramingHandler::read(wangle::HandlerContext<std::pair<std::unique_ptr<folly::IOBuf, std::default_delete<folly::IOBuf> >, std::unique_ptr<apache::thrift::transport::THeader, std::default_delete<apache::thrift::transport::THeader> > >, std::unique_ptr<folly::IOBuf, std::default_delete<folly::IOBuf> > >*, folly::IOBufQueue&) ()
#19 0x0000000004336d0d in non-virtual thunk to wangle::ContextImpl<apache::thrift::FramingHandler>::read(folly::IOBufQueue&) ()
#20 0x0000000004336bf0 in wangle::ContextImpl<apache::thrift::TAsyncTransportHandler>::fireRead(folly::IOBufQueue&) ()
#21 0x0000000004646e9f in folly::AsyncSocket::handleRead() ()
#22 0x000000000463dfa0 in folly::AsyncSocket::ioReady(unsigned short) ()
#23 0x000000000472ddef in ?? ()
#24 0x000000000472e70f in event_base_loop ()
#25 0x0000000004657675 in folly::EventBase::loopBody(int, bool) ()
#26 0x0000000004658246 in folly::EventBase::loop() ()
#27 0x0000000004659cf6 in folly::EventBase::loopForever() ()
#28 0x00000000045e3929 in folly::IOThreadPoolExecutor::threadRun(std::shared_ptr<folly::ThreadPoolExecutor::Thread>) ()
#29 0x00000000045f14f9 in void folly::detail::function::FunctionTraits<void ()>::callBig<std::_Bind<void (folly::ThreadPoolExecutor::*(folly::ThreadPoolExecutor*, std::shared_ptr<folly::ThreadPoolExecutor::Thread>))(std::shared_ptr<folly::ThreadPoolExecutor::Thread>)> >(folly::detail::function::Data&) ()
#30 0x0000000002a6b880 in folly::detail::function::FunctionTraits<void ()>::operator()() (this=0x7fc8c16072b0) at /data/src/wwl/nebula/build/third-party/install/include/folly/Function.h:400
#31 0x0000000002a7bb92 in folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}::operator()() (__closure=0x7fc8c16072b0) at /data/src/wwl/nebula/build/third-party/install/include/folly/executors/thread_factory/NamedThreadFactory.h:40
#32 0x0000000002a8d478 in std::__invoke_impl<void, folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}>(std::__invoke_other, folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}&&) (__f=...) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/bits/invoke.h:60
#33 0x0000000002a86b90 in std::__invoke<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}>(folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}&&) (__fn=...) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/bits/invoke.h:95
#34 0x0000000002ad23de in std::thread::_Invoker<std::tuple<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}> >::_M_invoke<0ul>(std::_Index_tuple<0ul>) (this=0x7fc8c16072b0) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/thread:234
#35 0x0000000002ad18d3 in std::thread::_Invoker<std::tuple<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}> >::operator()() (this=0x7fc8c16072b0) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/thread:243
#36 0x0000000002aceb70 in std::thread::_State_impl<std::thread::_Invoker<std::tuple<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}> > >::_M_run() (this=0x7fc8c16072a0) at /data/vesoft/toolset/gcc/7.5.0/include/c++/7.5.0/thread:186
#37 0x0000000004bb505f in execute_native_thread_routine ()
#38 0x00007fc8e11ecea5 in start_thread () from /lib64/libpthread.so.0
#39 0x00007fc8e0f159fd in clone () from /lib64/libc.so.6
```

卧槽，这哥们，folly 调度它去处理 heartbeat 请求，
然后它卡在 /root/nebula-workspace/nebula/src/kvstore/raftex/RaftPart.cpp:1650 上了，
1650 这行代码正要获取 raftLock_ 锁，raft 完美死锁了：

```c++
void RaftPart::processHeartbeatRequest(const cpp2::HeartbeatRequest& req,
                                       cpp2::HeartbeatResponse& resp) {
  LOG_IF(INFO, FLAGS_trace_raft) << idStr_ << "Received heartbeat"
                                 << ": GraphSpaceId = " << req.get_space()
                                 << ", partition = " << req.get_part()
                                 << ", leaderIp = " << req.get_leader_addr()
                                 << ", leaderPort = " << req.get_leader_port()
                                 << ", current_term = " << req.get_current_term()
                                 << ", lastLogId = " << req.get_last_log_id()
                                 << ", committedLogId = " << req.get_committed_log_id()
                                 << ", lastLogIdSent = " << req.get_last_log_id_sent()
                                 << ", lastLogTermSent = " << req.get_last_log_term_sent()
                                 << ", local lastLogId = " << lastLogId_
                                 << ", local lastLogTerm = " << lastLogTerm_
                                 << ", local committedLogId = " << committedLogId_
                                 << ", local current term = " << term_;
  std::lock_guard<std::mutex> g(raftLock_);
 
  resp.set_current_term(term_);
  resp.set_leader_addr(leader_.host);
  resp.set_leader_port(leader_.port);
  resp.set_committed_log_id(committedLogId_);
  resp.set_last_log_id(lastLogId_ < committedLogId_ ? committedLogId_ : lastLogId_);
  resp.set_last_log_term(lastLogTerm_);
 
  // Check status
  if (UNLIKELY(status_ == Status::STOPPED)) {
    VLOG(2) << idStr_ << "The part has been stopped, skip the request";
    resp.set_error_code(cpp2::ErrorCode::E_BAD_STATE);
    return;
  }
...
```
