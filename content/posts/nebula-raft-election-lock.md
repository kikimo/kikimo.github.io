---
title: "Nebula Raft 中的选举死锁"
date: 2022-01-09T19:05:51+08:00
draft: true
---

最近碰到 Nebula storage 节点无法选举的问题，
集体场景是：在一个七节点的集群中挂了三个节点，
按照预期剩下的四个节点应该能选出新的 leader 让继续提供服务，
但是我们观察到这四个节点选不出 leader 了：

![noleader](/images/raft-election/noleader.jpg)

我们筛选出没挂掉的四个节点：

```txt

# sh show.sh  | grep storaged | grep -v T
root     1840662  234  0.0 1464340 191056 ?      Ssl  14:42  51:20 /root/src/nebula/build/bin/nebula-storaged --flagfile /data/src/ntest/test/etc/nebula-storaged.conf --pid_file /data/src/ntest/test/pids/nebula-storaged.pid.0 --meta_server_addrs 192.168.15.12:9559 --heartbeat_interval_secs 1 --raft_heartbeat_interval_secs 1 --wal_ttl 259200 --clean_wal_interval_secs 259200 --minloglevel 2 --log_dir /data/src/ntest/test/logs/storaged.0 --local_ip 192.168.15.12 --port 54773 --ws_http_port 55577 --ws_h2_port 47925 --data_path /data/src/ntest/test/data/storaged.0
root     1840954  960  0.0 1473044 198408 ?      Ssl  14:42 208:39 /root/src/nebula/build/bin/nebula-storaged --flagfile /data/src/ntest/test/etc/nebula-storaged.conf --pid_file /data/src/ntest/test/pids/nebula-storaged.pid.2 --meta_server_addrs 192.168.15.12:9559 --heartbeat_interval_secs 1 --raft_heartbeat_interval_secs 1 --wal_ttl 259200 --clean_wal_interval_secs 259200 --minloglevel 2 --log_dir /data/src/ntest/test/logs/storaged.2 --local_ip 192.168.15.12 --port 39619 --ws_http_port 38621 --ws_h2_port 44865 --data_path /data/src/ntest/test/data/storaged.2
root     1841100  396  0.0 1476116 185236 ?      Ssl  14:43  85:48 /root/src/nebula/build/bin/nebula-storaged --flagfile /data/src/ntest/test/etc/nebula-storaged.conf --pid_file /data/src/ntest/test/pids/nebula-storaged.pid.3 --meta_server_addrs 192.168.15.12:9559 --heartbeat_interval_secs 1 --raft_heartbeat_interval_secs 1 --wal_ttl 259200 --clean_wal_interval_secs 259200 --minloglevel 2 --log_dir /data/src/ntest/test/logs/storaged.3 --local_ip 192.168.15.12 --port 48139 --ws_http_port 51591 --ws_h2_port 54371 --data_path /data/src/ntest/test/data/storaged.3
root     1841392  414  0.0 1467924 200676 ?      Ssl  14:43  88:54 /root/src/nebula/build/bin/nebula-storaged --flagfile /data/src/ntest/test/etc/nebula-storaged.conf --pid_file /data/src/ntest/test/pids/nebula-storaged.pid.5 --meta_server_addrs 192.168.15.12:9559 --heartbeat_interval_secs 1 --raft_heartbeat_interval_secs 1 --wal_ttl 259200 --clean_wal_interval_secs 259200 --minloglevel 2 --log_dir /data/src/ntest/test/logs/storaged.5 --local_ip 192.168.15.12 --port 33123 --ws_http_port 46329 --ws_h2_port 51909 --data_path /data/src/ntest/test/data/storaged.5
```

确定这几个节点的服务和端口:

```txt
storage.0 : 54773
storage.2 : 39619
storage.3 : 48139
storage.5 : 33123
```

我们同时对这四个节点的日志 tail -f，录下某个瞬间的截图，我们来详细分析这四个节点的选举过程：

![election log](/images/raft-election/election-log.jpg)

从日志中我们可以看出 storage.0 拒绝了 storage.5 的 vote request，原因有两个:

1. storage.0 投票给其他节点了
2. stroage.5 上的 term 远远落后于其他节点，日志上我们看到 storage.5 的 term 是 1834，而 storage.2 的 term 则是 1967

storage.5 的投票请求被拒绝的本质原因还是由于他的 term 比其他节点小。
同样的原因，stoage.2 和 storage.3 也拒绝了 strage.5 的投票请求。
由此可见 storage.5 的 term 如果一直都落后其他节点的话，那么它就不可能当选 leader。
再来看 storage.5，从日志上我们可以看出 storage.5 的 log 比其他三个节点都新，
所以他拒绝了所有其他节点的投票请求，那么现在问题就来了：

1. 要当选 leader 需要取得四个投票
2. storage.5 不会投票给任何其他节点，因为它上面的 log 最新，所以理论上只有 storage.5 才有当选 leader 的可能
3. storage.5 上的 term 看起来永远都追不上 其他节点的 term？为啥

我们知道，如果第三点成立，那么 raft 集群就就会一直陷入选举死循环了。
storage.5 的 term 为什么一直落后呢，我们看 nebula raft 中 RequestForVote RPC 的实现流程：

```cpp
void RaftPart::processAskForVoteRequest(const cpp2::AskForVoteRequest& req,
                                        cpp2::AskForVoteResponse& resp) {
  LOG(ERROR) << idStr_ << "Recieved a VOTING request"
            << ": space = " << req.get_space() << ", partition = " << req.get_part()
            << ", candidateAddr = " << req.get_candidate_addr() << ":" << req.get_candidate_port()
            << ", term = " << req.get_term() << ", lastLogId = " << req.get_last_log_id()
            << ", lastLogTerm = " << req.get_last_log_term();
 
  std::lock_guard<std::mutex> g(raftLock_);
 
  // Make sure the partition is running
  if (UNLIKELY(status_ == Status::STOPPED)) {
    LOG(ERROR) << idStr_ << "The part has been stopped, skip the request";
    resp.set_error_code(cpp2::ErrorCode::E_BAD_STATE);
    return;
  }
 
  if (UNLIKELY(status_ == Status::STARTING)) {
    LOG(ERROR) << idStr_ << "The partition is still starting";
    resp.set_error_code(cpp2::ErrorCode::E_NOT_READY);
    return;
  }
 
  if (UNLIKELY(status_ == Status::WAITING_SNAPSHOT)) {
    LOG(ERROR) << idStr_ << "The partition is still waiting snapshot";
    resp.set_error_code(cpp2::ErrorCode::E_WAITING_SNAPSHOT);
    return;
  }
 
  LOG(ERROR) << idStr_ << "The partition currently is a " << roleStr(role_) << ", lastLogId "
            << lastLogId_ << ", lastLogTerm " << lastLogTerm_ << ", committedLogId "
            << committedLogId_ << ", term " << term_;
  if (role_ == Role::LEARNER) {
    resp.set_error_code(cpp2::ErrorCode::E_BAD_ROLE);
    return;
  }
 
  auto candidate = HostAddr(req.get_candidate_addr(), req.get_candidate_port());
  // Check term id
  auto term = term_;
  if (req.get_term() <= term) {
    LOG(ERROR) << idStr_
              << (role_ == Role::CANDIDATE ? "The partition is currently proposing term "
                                           : "The partition currently is on term ")
              << term
              << ". The term proposed by the candidate is"
                 " no greater, so it will be rejected";
    resp.set_error_code(cpp2::ErrorCode::E_TERM_OUT_OF_DATE);
    return;
  }
 
  // Check the last term to receive a log
  if (req.get_last_log_term() < lastLogTerm_) {
    LOG(ERROR) << idStr_ << "The partition's last term to receive a log is " << lastLogTerm_
              << ", which is newer than the candidate's log " << req.get_last_log_term()
              << ". So the candidate will be rejected";
    resp.set_error_code(cpp2::ErrorCode::E_TERM_OUT_OF_DATE);
    return;
  }
 
  if (req.get_last_log_term() == lastLogTerm_) {
    // Check last log id
    if (req.get_last_log_id() < lastLogId_) {
      LOG(ERROR) << idStr_ << "The partition's last log id is " << lastLogId_
                << ". The candidate's last log id " << req.get_last_log_id()
                << " is smaller, so it will be rejected, candidate is " << candidate;
      resp.set_error_code(cpp2::ErrorCode::E_LOG_STALE);
      return;
    }
  }
 
  // If we have voted for somebody, we will reject other candidates under the
  // proposedTerm.
  if (votedAddr_ != HostAddr("", 0)) {
    if (proposedTerm_ > req.get_term() ||
        (proposedTerm_ == req.get_term() && votedAddr_ != candidate)) {
      LOG(ERROR) << idStr_ << "We have voted " << votedAddr_ << " on term " << proposedTerm_
                << ", so we should reject the candidate " << candidate << " request on term "
                << req.get_term();
      resp.set_error_code(cpp2::ErrorCode::E_TERM_OUT_OF_DATE);
      return;
    }
  }
 
  auto code = checkPeer(candidate);
  if (code != cpp2::ErrorCode::SUCCEEDED) {
    resp.set_error_code(code);
    return;
  }
  // Ok, no reason to refuse, we will vote for the candidate
  LOG(ERROR) << idStr_ << "The partition will vote for the candidate " << candidate;
  resp.set_error_code(cpp2::ErrorCode::SUCCEEDED);
 
  // Before change role from leader to follower, check the logs locally.
  if (role_ == Role::LEADER && wal_->lastLogId() > lastLogId_) {
    LOG(ERROR) << idStr_ << "There are some logs up to " << wal_->lastLogId()
              << " i did not commit when i was leader, rollback to " << lastLogId_;
    wal_->rollbackToLog(lastLogId_);
  }
  if (role_ == Role::LEADER) {
    bgWorkers_->addTask([self = shared_from_this(), term] { self->onLostLeadership(term); });
  }
  role_ = Role::FOLLOWER;
  votedAddr_ = candidate;
  proposedTerm_ = req.get_term();
  leader_ = HostAddr("", 0);
 
  // Reset the last message time
  lastMsgRecvDur_.reset();
  weight_ = 1;
  isBlindFollower_ = false;
  return;
}
```

看 1293 - 1310 这几行代码，
处理选举请求的时候如果 candidate 的 log 比自己的 log 旧，
raft 会直接拒绝这个请求。
这个操作逻辑上没问题，但是 Raft 论文里要求一个 Raft 实例一旦遇到比自己 term 大的请求要立马 update 自己的 term，
这个函数里执行这部操作了吗？
显然没有，判断日志比自己旧后就直接 return 了。
这会导致 storage.5 上的 term 几乎永远比其他节点小，
集群这这个状态下就永远无法选主。
为什么我要说几乎？
因为如果 storage.5 上的 term 和其他节点的 term 差距不是非常大，
经过长运行活着发生什么奇迹后它的 term 还有可能会追上来，
那时候系统就能选出 leader。
比如 raft 选举跑了一晚上没 leader，
第二天我 gdb attach 到几个节点 debug  一会儿再退出来系统就奇迹般的选出 leader 节。
