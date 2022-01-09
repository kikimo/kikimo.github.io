---
title: "Nebula Raft Appendlog 死锁问题分析"
date: 2022-01-09T17:43:08+08:00
draft: false
---

Nebula 在高并发的情况下整个集群经常会陷入不可用的状态，
在这种状态下不管你执行什么操作系统都只给你返回一个 leader change 的错误信息。
storage 的日志中我们可以看到大量如下报错信息：

```txt
W1019 08:26:21.220441 539751 RaftPart.cpp:601] [Port: 50944, Space: 3, Part: 1] The appendLog buffer is full. Please slow down the log appending rate.replicatingLogs_ :0
W1019 08:26:54.569221 539751 RaftPart.cpp:601] [Port: 50944, Space: 3, Part: 1] The appendLog buffer is full. Please slow down the log appending rate.replicatingLogs_ :0
W1019 08:27:27.919421 539751 RaftPart.cpp:601] [Port: 50944, Space: 3, Part: 1] The appendLog buffer is full. Please slow down the log appending rate.replicatingLogs_ :0
W1019 08:28:01.268051 539751 RaftPart.cpp:601] [Port: 50944, Space: 3, Part: 1] The appendLog buffer is full. Please slow down the log appending rate.replicatingLogs_ :0
W1019 08:28:34.615942 539751 RaftPart.cpp:601] [Port: 50944, Space: 3, Part: 1] The appendLog buffer is full. Please slow down the log appending rate.replicatingLogs_ :0
```

如果我们跟进 storage 的代码里就会发现，
`replicatingLogs_ `这个变量表示当前 Raft leader 是否正在复制日志到其他节点。
这个报错信息诡异的地方就在于，一方面系统告诉你 raft append log 缓冲已经满了，
另一方面 raft leader 又没在复制日志。

在`bool RaftPart::checkAppendLogResult(AppendLogResult res)`这个方法里我们可以看到，
它在把`replicatingLogs_`变量设置为`false`前释放`logsLock_`锁。
那么在`logsLock_`锁释放且`replicatingLogs = false`的这个间隙
系统可以一直 appendlog 直到 appendlog 的缓冲区满了、`bufferOverFlow`被设置为`true`。

```cpp
bool RaftPart::checkAppendLogResult(AppendLogResult res) { 
  if (res != AppendLogResult::SUCCEEDED) { 
    { 
      std::lock_guard<std::mutex> lck(logsLock_); 
      logs_.clear(); 
      cachingPromise_.setValue(res); 
      cachingPromise_.reset(); 
      bufferOverFlow_ = false; 
    } 
    sendingPromise_.setValue(res); 
    replicatingLogs_ = false; 
    return false; 
  } 
  return true; 
} 
```

这个操作是灾难性的，因为，在
`folly::Future<AppendLogResult> RaftPart::appendLogAsync()`
中我们看到一旦`bufferOverFlow`被设置为`true`，`appendLogAsync()`会立即返回报错，
同时，因为`replicatingLogs = false`系统中不会再启动任务消费缓冲区中的 Raft log，
整个服务就这样死锁了：

```cpp
 folly::Future<AppendLogResult> RaftPart::appendLogAsync(ClusterID source, 
                                                         LogType logType, 
                                                         std::string log, 
                                                         AtomicOp op) { 
   if (blocking_) { 
     // No need to block heartbeats and empty log. 
     if ((logType == LogType::NORMAL && !log.empty()) || logType == LogType::ATOMIC_OP) { 
       return AppendLogResult::E_WRITE_BLOCKING; 
     } 
   } 
  
   LogCache swappedOutLogs; 
   auto retFuture = folly::Future<AppendLogResult>::makeEmpty(); 
  
   if (bufferOverFlow_) { 
     LOG_EVERY_N(WARNING, 100) << idStr_ 
                               << "The appendLog buffer is full." 
                                  " Please slow down the log appending rate." 
                               << "replicatingLogs_ :" << replicatingLogs_; 
     return AppendLogResult::E_BUFFER_OVERFLOW; 
   } 
   { 
     std::lock_guard<std::mutex> lck(logsLock_); 
  
     VLOG(2) << idStr_ << "Checking whether buffer overflow"; 
  
     if (logs_.size() >= FLAGS_max_batch_size) { 
       // Buffer is full 
       LOG(WARNING) << idStr_ 
                    << "The appendLog buffer is full." 
                       " Please slow down the log appending rate." 
                    << "replicatingLogs_ :" << replicatingLogs_; 
       bufferOverFlow_ = true; 
       return AppendLogResult::E_BUFFER_OVERFLOW; 
     } 
  
     VLOG(2) << idStr_ << "Appending logs to the buffer"; 
  
     // Append new logs to the buffer 
     DCHECK_GE(source, 0); 
     logs_.emplace_back(source, logType, std::move(log), std::move(op)); 
     switch (logType) { 
       case LogType::ATOMIC_OP: 
         retFuture = cachingPromise_.getSingleFuture(); 
         break; 
       case LogType::COMMAND: 
         retFuture = cachingPromise_.getAndRollSharedFuture(); 
         break; 
       case LogType::NORMAL: 
         retFuture = cachingPromise_.getSharedFuture(); 
         break; 
     } 
  
     bool expected = false; 
     if (replicatingLogs_.compare_exchange_strong(expected, true)) { 
       // We need to send logs to all followers 
       VLOG(2) << idStr_ << "Preparing to send AppendLog request"; 
       sendingPromise_ = std::move(cachingPromise_); 
       cachingPromise_.reset(); 
       std::swap(swappedOutLogs, logs_); 
       bufferOverFlow_ = false; 
     } else { 
       VLOG(2) << idStr_ << "Another AppendLogs request is ongoing, just return"; 
       return retFuture; 
     } 
   } 
...
```

