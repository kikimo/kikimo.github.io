---
title: "Nebula Raft 中的数据不一致问题"
date: 2022-01-09T19:22:15+08:00
draft: false
---

最近发现一个 Nebula Raft 中数据不一致的问题。
我们构造了一个五存储节点的 Nebula 集群，
通过 storage 的 raft kv 接口往 Raft 写数据。
在写了 48w 左右的数据后我们停止数据写入并检测 Raft Wal 和 RocksDB 中的数据。
在这两个点上都发现了数据不一致的问题。
五个 storage 节点中，store2、store4、store5 的数据是一致的，
store1、store3 的数据是一致，
但是这两个集合存在部分数据不一致的情况。
从以下 WAL 的对比结果中我们可以发现，store1、store2 有 19 条 log 不一致：

```txt
1c1
< /data/src/nebula-cluster/data/data/store1/nebula/1/wal/1
---
> /data/src/nebula-cluster/data/data/store2/nebula/1/wal/1
293702,293720c293702,293720
< log index: 293701, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293702, term: 694, logsz: 57, cluster_id: 0, walfile:
< log index: 293703, term: 694, logsz: 57, cluster_id: 0, walfile:
< log index: 293704, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293705, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293706, term: 694, logsz: 57, cluster_id: 0, walfile:
< log index: 293707, term: 694, logsz: 57, cluster_id: 0, walfile:
< log index: 293708, term: 694, logsz: 57, cluster_id: 0, walfile:
< log index: 293709, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293710, term: 694, logsz: 57, cluster_id: 0, walfile:
< log index: 293711, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293712, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293713, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293714, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293715, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293716, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293717, term: 694, logsz: 57, cluster_id: 0, walfile:
< log index: 293718, term: 694, logsz: 55, cluster_id: 0, walfile:
< log index: 293719, term: 695, logsz: 0, cluster_id: 0, walfile:
---
> log index: 293701, term: 696, logsz: 53, cluster_id: 0, walfile:
> log index: 293702, term: 696, logsz: 55, cluster_id: 0, walfile:
> log index: 293703, term: 696, logsz: 59, cluster_id: 0, walfile:
> log index: 293704, term: 696, logsz: 57, cluster_id: 0, walfile:
> log index: 293705, term: 696, logsz: 55, cluster_id: 0, walfile:
> log index: 293706, term: 696, logsz: 57, cluster_id: 0, walfile:
> log index: 293707, term: 696, logsz: 57, cluster_id: 0, walfile:
> log index: 293708, term: 696, logsz: 59, cluster_id: 0, walfile:
> log index: 293709, term: 696, logsz: 55, cluster_id: 0, walfile:
> log index: 293710, term: 696, logsz: 57, cluster_id: 0, walfile:
> log index: 293711, term: 696, logsz: 55, cluster_id: 0, walfile:
> log index: 293712, term: 696, logsz: 55, cluster_id: 0, walfile:
> log index: 293713, term: 696, logsz: 55, cluster_id: 0, walfile:
> log index: 293714, term: 696, logsz: 57, cluster_id: 0, walfile:
> log index: 293715, term: 696, logsz: 57, cluster_id: 0, walfile:
> log index: 293716, term: 696, logsz: 57, cluster_id: 0, walfile:
> log index: 293717, term: 696, logsz: 57, cluster_id: 0, walfile:
> log index: 293718, term: 696, logsz: 55, cluster_id: 0, walfile:
> log index: 293719, term: 696, logsz: 53, cluster_id: 0, walfile:
```

RocksDB 中的数据不一致分布情况和 Raft WAL 几乎是一一对应。
也是 store2、store4、store5 和 store1、store3 中的数据不一致。
仍然取 store1 和 store2 中的 RocksDB 数据做对比，
我们发现非常有意思的一个情况：
store1、store2 中不一致的数据条目是 18，
和 Raft WAL 中不一致的数据条目 19 非常接近，
而相差的一条数据似乎和 Raft WAL 中的一条 log szie 为 0 的数据有关。
具体的差异情况是 store1 中有 18 条数据 store2 没有，
store2 中也有 18 条数据 store1 不存在，
store2 又比 store1 多了一条数据。详情如下：

```txt
comparing /Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data
size mismatch: 489347, 489348
/Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-1-12197-340'
b'\x06\x01\x00\x00key-1-11350-767'
b'\x06\x01\x00\x00key-1-12553-44'
b'\x06\x01\x00\x00key-1-10677-952'
b'\x06\x01\x00\x00key-1-13514-912'
b'\x06\x01\x00\x00key-1-9430-782'
b'\x06\x01\x00\x00key-1-18022-735'
b'\x06\x01\x00\x00key-1-7029-104'
b'\x06\x01\x00\x00key-1-4530-867'
b'\x06\x01\x00\x00key-1-8658-248'
b'\x06\x01\x00\x00key-1-8489-415'
b'\x06\x01\x00\x00key-1-2345-956'
b'\x06\x01\x00\x00key-1-8213-336'
b'\x06\x01\x00\x00key-1-8330-687'
b'\x06\x01\x00\x00key-1-9470-108'
b'\x06\x01\x00\x00key-0-62674-143'
b'\x06\x01\x00\x00key-1-12613-884'
b'\x06\x01\x00\x00key-1-8860-507'
/Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-1-9504-429'
b'\x06\x01\x00\x00key-1-15925-489'
b'\x06\x01\x00\x00key-1-17467-978'
b'\x06\x01\x00\x00key-1-14189-663'
b'\x06\x01\x00\x00key-1-6414-170'
b'\x06\x01\x00\x00key-1-11835-136'
b'\x06\x01\x00\x00key-1-10409-874'
b'\x06\x01\x00\x00key-1-6672-385'
b'\x06\x01\x00\x00key-1-17840-561'
b'\x06\x01\x00\x00key-1-13118-1010'
b'\x06\x01\x00\x00key-1-7707-630'
b'\x06\x01\x00\x00key-1-5606-677'
b'\x06\x01\x00\x00key-1-10107-197'
b'\x06\x01\x00\x00key-0-64103-1001'
b'\x06\x01\x00\x00key-1-6373-99'
b'\x06\x01\x00\x00key-1-940-285'
b'\x06\x01\x00\x00key-1-10802-736'
b'\x06\x01\x00\x00key-1-7087-647'
b'\x06\x01\x00\x00key-1-3020-441'
diff 1-2: []
```

这是一个相当蛋疼的问题。
但幸运的是我们发开 Mozilla RR 录制的时候也可以把这个问题复现出来。
即便如此我还是盯着屏幕 debug 了一天后才定位到问题的根因。
总得来说导致数据不一致的问题有两点：

1. Nebula Raft 中处理处理节点身份转换的逻辑有问题
2. Nebula Raft log match 实现逻辑有问题

先说 Raft log match 的问题。
以下是 RaftPart.cpp 中处理选举的请求的函数：

```cpp
void RaftPart::processAskForVoteRequest(const cpp2::AskForVoteRequest& req,
                                        cpp2::AskForVoteResponse& resp) {
  LOG(INFO) << idStr_ << "Received a VOTING request"
            << ": space = " << req.get_space() << ", partition = " << req.get_part()
            << ", candidateAddr = " << req.get_candidate_addr() << ":" << req.get_candidate_port()
            << ", term = " << req.get_term() << ", lastLogId = " << req.get_last_log_id()
            << ", lastLogTerm = " << req.get_last_log_term()
            << ", isPreVote = " << req.get_is_pre_vote();
 
  std::lock_guard<std::mutex> g(raftLock_);
 
  // Make sure the partition is running
  if (UNLIKELY(status_ == Status::STOPPED)) {
    LOG(INFO) << idStr_ << "The part has been stopped, skip the request";
    resp.set_error_code(cpp2::ErrorCode::E_BAD_STATE);
    return;
  }
 
  if (UNLIKELY(status_ == Status::STARTING)) {
    LOG(INFO) << idStr_ << "The partition is still starting";
    resp.set_error_code(cpp2::ErrorCode::E_NOT_READY);
    return;
  }
 
  if (UNLIKELY(status_ == Status::WAITING_SNAPSHOT)) {
    LOG(INFO) << idStr_ << "The partition is still waiting snapshot";
    resp.set_error_code(cpp2::ErrorCode::E_WAITING_SNAPSHOT);
    return;
  }
 
  LOG(INFO) << idStr_ << "The partition currently is a " << roleStr(role_) << ", lastLogId "
            << lastLogId_ << ", lastLogTerm " << lastLogTerm_ << ", committedLogId "
            << committedLogId_ << ", term " << term_;
  if (role_ == Role::LEARNER) {
    resp.set_error_code(cpp2::ErrorCode::E_BAD_ROLE);
    return;
  }
 
  auto candidate = HostAddr(req.get_candidate_addr(), req.get_candidate_port());
  auto code = checkPeer(candidate);
  if (code != cpp2::ErrorCode::SUCCEEDED) {
    resp.set_error_code(code);
    return;
  }
 
  // Check term id
  if (req.get_term() < term_) {
    LOG(INFO) << idStr_ << "The partition currently is on term " << term_
              << ", the term proposed by the candidate is " << req.get_term()
              << ", so it will be rejected";
    resp.set_error_code(cpp2::ErrorCode::E_TERM_OUT_OF_DATE);
    return;
  }
 
  auto oldTerm = term_;
  // req.get_term() >= term_, we won't update term in prevote
  if (!req.get_is_pre_vote()) {
    term_ = req.get_term();
  }
 
  // Check the last term to receive a log
  if (req.get_last_log_term() < lastLogTerm_) {
    LOG(INFO) << idStr_ << "The partition's last term to receive a log is " << lastLogTerm_
              << ", which is newer than the candidate's log " << req.get_last_log_term()
              << ". So the candidate will be rejected";
    resp.set_error_code(cpp2::ErrorCode::E_TERM_OUT_OF_DATE);
    return;
  }
 
  if (req.get_last_log_term() == lastLogTerm_) {
    // Check last log id
    if (req.get_last_log_id() < lastLogId_) {
      LOG(INFO) << idStr_ << "The partition's last log id is " << lastLogId_
                << ". The candidate's last log id " << req.get_last_log_id()
                << " is smaller, so it will be rejected, candidate is " << candidate;
      resp.set_error_code(cpp2::ErrorCode::E_LOG_STALE);
      return;
    }
  }
...
```

注意 1309 - 1318 这几行代码，
在 1309 这行代码中，
一个 leader 的 term 可能直接被 update 了，
1318 行判断日志的情况后就直接返回了，
这会导致本来应该被转换成 follower 的 leader 直接变成新一轮 term 的 leader，
这是导致 Raft 数据不一致最直接的原因。
这个不应该出现的 leader 不会重新初始化 Host peer 的 log 下标，
又 Host.cpp 在判断 log match 上的逻辑不完备导致本来应该被 rollback 的 log 直接被 commit 了。
从 Mozillar RR 的 replay 过程中我们观察到原本应当变成 Follower 的节点直接当选下一轮 leader 的整个过程：

```txt
(rr) b 1308
Breakpoint 10 at 0x4077555: file /root/src/nebula/src/kvstore/raftex/RaftPart.cpp, line 1308.
(rr) reverse-cont
Continuing.
 
Thread 3 hit Hardware watchpoint 2: *(int*) 0x7f5e4ed00208
 
Old value = 84
New value = 82
0x0000000004077581 in nebula::raftex::RaftPart::processAskForVoteRequest (this=0x7f5e4ed00010, req=..., resp=...)
    at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1309
1309        term_ = req.get_term();
1: lastLogIdSent_ = <error: There is no member or method named lastLogIdSent_.>
2: lastLogTermSent_ = <error: There is no member or method named lastLogTermSent_.>
(rr) n
 
Thread 3 hit Hardware watchpoint 2: *(int*) 0x7f5e4ed00208
 
Old value = 82
New value = 84
nebula::raftex::RaftPart::processAskForVoteRequest (this=0x7f5e4ed00010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1313
1313      if (req.get_last_log_term() < lastLogTerm_) {
1: lastLogIdSent_ = <error: There is no member or method named lastLogIdSent_.>
2: lastLogTermSent_ = <error: There is no member or method named lastLogTermSent_.>
(rr) n
1314        LOG(INFO) << idStr_ << "The partition's last term to receive a log is " << lastLogTerm_
1: lastLogIdSent_ = <error: There is no member or method named lastLogIdSent_.>
2: lastLogTermSent_ = <error: There is no member or method named lastLogTermSent_.>
(rr) p lastLogTerm_
$29 = 82
(rr) p req.get_last_log_term()
$30 = 81
(rr)
```

在这次调试中，我们观察到的 RocksDB 数据不一致情况如下：

```txt
comparing /Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data
size mismatch: 56093, 56091
/Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-320-317'
b'\x06\x01\x00\x00key-0-1467-266'
/Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data missing keys:
diff 1-2: []
 
comparing /Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store3/nebula/1/data
size mismatch: 56093, 56092
/Users/wenlinwu/src/toss_integration/data/store3/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-1467-266'
/Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data missing keys:
diff 1-3: []
 
comparing /Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store4/nebula/1/data
/Users/wenlinwu/src/toss_integration/data/store4/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-1467-266'
/Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-458-122'
diff 1-4: [{'key': b'\x04\x01\x00\x00\x01\x00\x00\x00', 'val1': b'\xea\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00', 'val2': b'\xe9\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00'}]
 
comparing /Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store5/nebula/1/data
/Users/wenlinwu/src/toss_integration/data/store5/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-1467-266'
/Users/wenlinwu/src/toss_integration/data/store1/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-1012-309'
diff 1-5: [{'key': b'\x04\x01\x00\x00\x01\x00\x00\x00', 'val1': b'\xea\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00', 'val2': b'\xe9\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00'}]
 
comparing /Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store3/nebula/1/data
size mismatch: 56091, 56092
/Users/wenlinwu/src/toss_integration/data/store3/nebula/1/data missing keys:
/Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-320-317'
diff 2-3: []
 
comparing /Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store4/nebula/1/data
size mismatch: 56091, 56093
/Users/wenlinwu/src/toss_integration/data/store4/nebula/1/data missing keys:
/Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-320-317'
b'\x06\x01\x00\x00key-0-458-122'
diff 2-4: [{'key': b'\x04\x01\x00\x00\x01\x00\x00\x00', 'val1': b'\xea\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00', 'val2': b'\xe9\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00'}]
 
comparing /Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store5/nebula/1/data
size mismatch: 56091, 56093
/Users/wenlinwu/src/toss_integration/data/store5/nebula/1/data missing keys:
/Users/wenlinwu/src/toss_integration/data/store2/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-320-317'
b'\x06\x01\x00\x00key-0-1012-309'
diff 2-5: [{'key': b'\x04\x01\x00\x00\x01\x00\x00\x00', 'val1': b'\xea\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00', 'val2': b'\xe9\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00'}]
 
comparing /Users/wenlinwu/src/toss_integration/data/store3/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store4/nebula/1/data
size mismatch: 56092, 56093
/Users/wenlinwu/src/toss_integration/data/store4/nebula/1/data missing keys:
/Users/wenlinwu/src/toss_integration/data/store3/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-458-122'
diff 3-4: [{'key': b'\x04\x01\x00\x00\x01\x00\x00\x00', 'val1': b'\xea\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00', 'val2': b'\xe9\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00'}]
 
comparing /Users/wenlinwu/src/toss_integration/data/store3/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store5/nebula/1/data
size mismatch: 56092, 56093
/Users/wenlinwu/src/toss_integration/data/store5/nebula/1/data missing keys:
/Users/wenlinwu/src/toss_integration/data/store3/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-1012-309'
diff 3-5: [{'key': b'\x04\x01\x00\x00\x01\x00\x00\x00', 'val1': b'\xea\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00', 'val2': b'\xe9\xe4\x00\x00\x00\x00\x00\x00`\x02\x00\x00\x00\x00\x00\x00'}]
 
comparing /Users/wenlinwu/src/toss_integration/data/store4/nebula/1/data to /Users/wenlinwu/src/toss_integration/data/store5/nebula/1/data
/Users/wenlinwu/src/toss_integration/data/store5/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-458-122'
/Users/wenlinwu/src/toss_integration/data/store4/nebula/1/data missing keys:
b'\x06\x01\x00\x00key-0-1012-309'
diff 4-5: []
```

从 RocksDB 的比较数据中我们可以看到 store1 相对 store2 多了两条 key:

```txt
b'\x06\x01\x00\x00key-0-320-317'
b'\x06\x01\x00\x00key-0-1467-266'
```

通过 diff store1 store2 的 wal 记录，我们发现 store1 和 store2 有两条 logsz  > 0 的日志内容不一致：

```txt
# diff s0 s1
1c1
< /data/src/nebula-cluster/data/data/store1/nebula/1/wal/1
---
> /data/src/nebula-cluster/data/data/store2/nebula/1/wal/1
12126c12126
< log index: 12125, term: 84, logsz: 53, cluster_id: 0, walfile:
---
> log index: 12125, term: 83, logsz: 0, cluster_id: 0, walfile:
37443c37443
< log index: 37442, term: 505, logsz: 0, cluster_id: 0, walfile:
---
> log index: 37442, term: 503, logsz: 0, cluster_id: 0, walfile:
37453c37453
< log index: 37452, term: 511, logsz: 0, cluster_id: 0, walfile:
---
> log index: 37452, term: 510, logsz: 0, cluster_id: 0, walfile:
37483c37483
< log index: 37482, term: 527, logsz: 55, cluster_id: 0, walfile:
---
> log index: 37482, term: 527, logsz: 0, cluster_id: 0, walfile:
```

store1 多出的两条记录：

```txt
< log index: 12125, term: 84, logsz: 53, cluster_id: 0, walfile:
< log index: 37482, term: 527, logsz: 55, cluster_id: 0, walfile:
```

通过 rr 的过程回放，我们定位到 store1 写入 key: key-0-320-317 的瞬间：

```txt
#0  nebula::kvstore::Part::commitLogs (this=0x7f6e87e17010, iter=..., wait=false) at /root/src/nebula/src/kvstore/Part.cpp:248
#1  0x0000000004079ab9 in nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f6e87e17010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1556
#2  0x00000000040c07b9 in nebula::raftex::RaftexService::appendLog (this=0x7f6eac5e0550, resp=..., req=...) at /root/src/nebula/src/kvstore/raftex/RaftexService.cpp:203
#3  0x00000000041814ca in nebula::raftex::cpp2::RaftexServiceSvIf::<lambda(nebula::raftex::cpp2::AppendLogResponse&)>::operator()(nebula::raftex::cpp2::AppendLogResponse &) const (__closure=0x7f6eb4633590, _return=...)
    at /root/src/nebula/build/src/interface/gen-cpp2/RaftexService.cpp:44
#4  0x0000000004185de9 in apache::thrift::detail::si::returning<nebula::raftex::cpp2::RaftexServiceSvIf::semifuture_appendLog(const nebula::raftex::cpp2::AppendLogRequest&)::<lambda(nebula::raftex::cpp2::AppendLogResponse&)> >(nebula::raftex::cpp2::RaftexServiceSvIf::<lambda(nebula::raftex::cpp2::AppendLogResponse&)> &&) (f=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1204
#5  0x0000000004182d63 in apache::thrift::detail::si::<lambda()>::operator()(void) const (this=0x7f6eb4633550) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1210
#6  0x000000000418bfa0 in folly::<lambda()>::operator()(void) (this=0x7f6eb4633488) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:608
#7  0x0000000004197f15 in folly::makeTryWithNoUnwrap<folly::makeSemiFutureWith(F&&) [with F = apache::thrift::detail::si::semifuture_returning(F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::semifuture_appendLog(const nebula::raftex::cpp2::AppendLogRequest&)::<lambda(nebula::raftex::cpp2::AppendLogResponse&)>]::<lambda()>]::<lambda()> >(folly::<lambda()> &&) (f=...) at /root/src/nebula/build/third-party/install/include/folly/Try-inl.h:244
#8  0x0000000004191ff6 in folly::makeTryWith<folly::makeSemiFutureWith(F&&) [with F = apache::thrift::detail::si::semifuture_returning(F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::semifuture_appendLog(const nebula::raftex::cpp2::AppendLogRequest&)::<lambda(nebula::raftex::cpp2::AppendLogResponse&)>]::<lambda()>]::<lambda()> >(folly::<lambda()> &&) (f=...) at /root/src/nebula/build/third-party/install/include/folly/Try-inl.h:270
#9  0x000000000418c005 in folly::makeSemiFutureWith<apache::thrift::detail::si::semifuture_returning(F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::semifuture_appendLog(const nebula::raftex::cpp2::AppendLogRequest&)::<lambda(nebula::raftex::cpp2::AppendLogResponse&)>]::<lambda()> >(apache::thrift::detail::si::<lambda()> &&) (func=...) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:608
#10 0x0000000004185e6a in apache::thrift::detail::si::semifuture<apache::thrift::detail::si::semifuture_returning(F&&) [with F = nebula::raftex::cpp2::RaftexServiceSvIf::semifuture_appendLog(const nebula::raftex::cpp2::AppendLogRequest&)::<lambda(nebula::raftex::cpp2::AppendLogResponse&)>]::<lambda()> >(apache::thrift::detail::si::<lambda()> &&) (f=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1198
#11 0x0000000004182dc1 in apache::thrift::detail::si::semifuture_returning<nebula::raftex::cpp2::RaftexServiceSvIf::semifuture_appendLog(const nebula::raftex::cpp2::AppendLogRequest&)::<lambda(nebula::raftex::cpp2::AppendLogResponse&)> >(nebula::raftex::cpp2::RaftexServiceSvIf::<lambda(nebula::raftex::cpp2::AppendLogResponse&)> &&) (f=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1210
#12 0x0000000004181518 in nebula::raftex::cpp2::RaftexServiceSvIf::semifuture_appendLog (this=0x7f6eac5e0550, p_req=...) at /root/src/nebula/build/src/interface/gen-cpp2/RaftexService.cpp:44
#13 0x0000000004181611 in nebula::raftex::cpp2::RaftexServiceSvIf::future_appendLog (this=0x7f6eac5e0550, p_req=...) at /root/src/nebula/build/src/interface/gen-cpp2/RaftexService.cpp:51
#14 0x00000000041816fd in nebula::raftex::cpp2::RaftexServiceSvIf::<lambda()>::operator()(void) const (__closure=0x7f6eb46337b0) at /root/src/nebula/build/src/interface/gen-cpp2/RaftexService.cpp:56
#15 0x0000000004186170 in folly::makeFutureWith<nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()> >(nebula::raftex::cpp2::RaftexServiceSvIf::<lambda()> &&) (func=...) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:1241
#16 0x0000000004182fd7 in apache::thrift::detail::si::makeFutureWithAndMaybeReschedule<nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()> >(apache::thrift::detail::si::CallbackBase &, nebula::raftex::cpp2::RaftexServiceSvIf::<lambda()> &&) (callback=..., f=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1240
#17 0x0000000004183232 in apache::thrift::detail::si::async_tm<nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog(std::unique_ptr<apache::thrift::HandlerCallback<nebula::raftex::cpp2::AppendLogResponse> >, const nebula::raftex::cpp2::AppendLogRequest&)::<lambda()> >(apache::thrift::ServerInterface *, apache::thrift::detail::si::CallbackPtr, nebula::raftex::cpp2::RaftexServiceSvIf::<lambda()> &&) (si=0x7f6eac5e0558, callback=..., f=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1263
#18 0x0000000004181789 in nebula::raftex::cpp2::RaftexServiceSvIf::async_tm_appendLog (this=0x7f6eac5e0550, callback=..., p_req=...) at /root/src/nebula/build/src/interface/gen-cpp2/RaftexService.cpp:55
#19 0x000000000418a8af in nebula::raftex::cpp2::RaftexServiceAsyncProcessor::process_appendLog<apache::thrift::CompactProtocolReader, apache::thrift::CompactProtocolWriter> (this=0x7f6ebb722720, req=..., serializedRequest=..., ctx=0x7f6e87e1ad10, eb=0x7f6eb2e61f00,
    tm=0x7f6ebf4151d0) at /root/src/nebula/build/src/interface/gen-cpp2/RaftexService.tcc:113
#20 0x000000000418e518 in apache::thrift::GeneratedAsyncProcessor::makeEventTaskForRequest<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::raftex::cpp2::RaftexServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::raftex::cpp2::RaftexServiceAsyncProcessor*, apache::thrift::Tile*)::{lambda(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>)#1}::operator()(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>) (this=0x7f6e87db41c0, rq=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/async/AsyncProcessor.h:698
#21 0x00000000041a618e in folly::detail::function::FunctionTraits<void (std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>)>::callBig<apache::thrift::GeneratedAsyncProcessor::makeEventTaskForRequest<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::raftex::cpp2::RaftexServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::raftex::cpp2::RaftexServiceAsyncProcessor*, apache::thrift::Tile*)::{lambda(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>)#1}>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>&&, folly::detail::function::Data&) (args#0=..., p=...) at /root/src/nebula/build/third-party/install/include/folly/Function.h:385
#22 0x0000000004557c62 in apache::thrift::EventTask::run() ()
#23 0x000000000418889b in apache::thrift::GeneratedAsyncProcessor::processInThread<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::raftex::cpp2::RaftexServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::raftex::cpp2::RaftexServiceAsyncProcessor*)::{lambda()#1}::operator()() const (this=0x7f6ebb774690)
    at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/async/AsyncProcessor.h:773
#24 0x000000000419991a in folly::detail::function::FunctionTraits<void ()>::callSmall<apache::thrift::GeneratedAsyncProcessor::processInThread<nebula::raftex::cpp2::RaftexServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::raftex::cpp2::RaftexServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::raftex::cpp2::RaftexServiceAsyncProcessor*)::{lambda()#1}>(folly::detail::function::Data&) (p=...) at /root/src/nebula/build/third-party/install/include/folly/Function.h:371
#25 0x000000000455d107 in virtual thunk to apache::thrift::concurrency::FunctionRunner::run() ()
#26 0x0000000004692503 in apache::thrift::concurrency::ThreadManager::Impl::Worker::run() ()
#27 0x000000000469615d in apache::thrift::concurrency::PthreadThread::threadMain(void*) ()
#28 0x00007f6ebfc75609 in start_thread (arg=<optimized out>) at pthread_create.c:477
#29 0x00007f6ebfb9c293 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
(rr) ls
Undefined command: "ls".  Try "help".
(rr) l
243             // Make the number of values are an even number
244             DCHECK_EQ((kvs.size() + 1) / 2, kvs.size() / 2);
245             for (size_t i = 0; i < kvs.size(); i += 2) {
246               VLOG(1) << "OP_MULTI_PUT " << folly::hexlify(kvs[i])
247                       << ", val = " << folly::hexlify(kvs[i + 1]);
248               auto code = batch->put(kvs[i], kvs[i + 1]);
249               if (code != nebula::cpp2::ErrorCode::SUCCEEDED) {
250                 LOG(ERROR) << idStr_ << "Failed to call WriteBatch::put()";
251                 return code;
252               }
(rr) p kvs[1]
$3 = (__gnu_cxx::__alloc_traits<std::allocator<folly::Range<char const*> >, folly::Range<char const*> >::value_type &) @0x7f6ebb7acb30: {static npos = <optimized out>, b_ = 0x7f6ebb8cc9e6 "value-0-320-317", e_ = 0x7f6ebb8cc9f5 ""}
(rr) p log
$4 = {static npos = <optimized out>, b_ = 0x7f6ebb8cc9c0 "\200W\202[}\001", e_ = 0x7f6ebb8cc9f5 ""}
(rr) p lastId
$5 = 12125
(rr) p lastTerm
$6 = 84
(rr)
```

可以看到 key-0-320-317 对应的刚好是 index 12125, term 84 的 wal log。相应的我们可以在 store5 中看到  key-0-320-317  的 put 请求：

```txt
Thread 5 hit Breakpoint 1, nebula::storage::PutProcessor::<lambda(auto:146&)>::operator()<const std::pair<int const, std::vector<nebula::KeyValue> > >(const std::pair<int const, std::vector<nebula::KeyValue, std::allocator<nebula::KeyValue> > > &) const (__closure=0x7f5e834d1590, value=...) at /root/src/nebula/src/storage/kv/PutProcessor.cpp:25
25            data.emplace_back(std::move(NebulaKeyUtils::kvKey(part, pair.key)), std::move(pair.value));
(rr) p pair.key
$1 = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f5e4cb27550 "key-0-320-317"}, _M_string_length = 13, {_M_local_buf = "key-0-320-317\000\000",
    _M_allocated_capacity = 3617284610653250923}}
(rr) bt
#0  nebula::storage::PutProcessor::<lambda(auto:146&)>::operator()<const std::pair<int const, std::vector<nebula::KeyValue> > >(const std::pair<int const, std::vector<nebula::KeyValue, std::allocator<nebula::KeyValue> > > &) const (__closure=0x7f5e834d1590, value=...)
    at /root/src/nebula/src/storage/kv/PutProcessor.cpp:25
#1  0x0000000002d7af23 in std::for_each<std::__detail::_Node_const_iterator<std::pair<int const, std::vector<nebula::KeyValue> >, false, false>, nebula::storage::PutProcessor::process(const nebula::storage::cpp2::KVPutRequest&)::<lambda(auto:146&)> >(std::__detail::_Node_const_iterator<std::pair<int const, std::vector<nebula::KeyValue, std::allocator<nebula::KeyValue> > >, false, false>, std::__detail::_Node_const_iterator<std::pair<int const, std::vector<nebula::KeyValue, std::allocator<nebula::KeyValue> > >, false, false>, nebula::storage::PutProcessor::<lambda(auto:146&)>) (__first=..., __last=..., __f=...) at /usr/include/c++/9/bits/stl_algo.h:3882
#2  0x0000000002d7ac7f in nebula::storage::PutProcessor::process (this=0x7f5e79d57040, req=...) at /root/src/nebula/src/storage/kv/PutProcessor.cpp:28
#3  0x0000000002c08b5b in nebula::storage::GraphStorageServiceHandler::future_put (this=0x7f5e652b9090, req=...) at /root/src/nebula/src/storage/GraphStorageServiceHandler.cpp:166
#4  0x0000000003497dcd in nebula::storage::cpp2::GraphStorageServiceSvIf::<lambda()>::operator()(void) const (__closure=0x7f5e834d17f0) at /root/src/nebula/build/src/interface/gen-cpp2/GraphStorageService.cpp:392
#5  0x00000000034abb14 in folly::makeFutureWith<nebula::storage::cpp2::GraphStorageServiceSvIf::async_tm_put(std::unique_ptr<apache::thrift::HandlerCallback<nebula::storage::cpp2::ExecResponse> >, const nebula::storage::cpp2::KVPutRequest&)::<lambda()> >(nebula::storage::cpp2::GraphStorageServiceSvIf::<lambda()> &&) (func=...) at /root/src/nebula/build/third-party/install/include/folly/futures/Future-inl.h:1241
#6  0x000000000349f838 in apache::thrift::detail::si::makeFutureWithAndMaybeReschedule<nebula::storage::cpp2::GraphStorageServiceSvIf::async_tm_put(std::unique_ptr<apache::thrift::HandlerCallback<nebula::storage::cpp2::ExecResponse> >, const nebula::storage::cpp2::KVPutRequest&)::<lambda()> >(apache::thrift::detail::si::CallbackBase &, nebula::storage::cpp2::GraphStorageServiceSvIf::<lambda()> &&) (callback=..., f=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1240
#7  0x000000000349fa94 in apache::thrift::detail::si::async_tm<nebula::storage::cpp2::GraphStorageServiceSvIf::async_tm_put(std::unique_ptr<apache::thrift::HandlerCallback<nebula::storage::cpp2::ExecResponse> >, const nebula::storage::cpp2::KVPutRequest&)::<lambda()> >(apache::thrift::ServerInterface *, apache::thrift::detail::si::CallbackPtr, nebula::storage::cpp2::GraphStorageServiceSvIf::<lambda()> &&) (si=0x7f5e652b9098, callback=..., f=...) at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/GeneratedCodeHelper.h:1263
#8  0x0000000003497e59 in nebula::storage::cpp2::GraphStorageServiceSvIf::async_tm_put (this=0x7f5e652b9090, callback=..., p_req=...) at /root/src/nebula/build/src/interface/gen-cpp2/GraphStorageService.cpp:391
#9  0x00000000034b1ab3 in nebula::storage::cpp2::GraphStorageServiceAsyncProcessor::process_put<apache::thrift::BinaryProtocolReader, apache::thrift::BinaryProtocolWriter> (this=0x7f5e864a9920, req=..., serializedRequest=..., ctx=0x7f5e79c9c610, eb=0x7f5e8649bf00,
    tm=0x7f5e864151d0) at /root/src/nebula/build/src/interface/gen-cpp2/GraphStorageService.tcc:1053
#10 0x00000000034c1b1e in apache::thrift::GeneratedAsyncProcessor::makeEventTaskForRequest<nebula::storage::cpp2::GraphStorageServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::storage::cpp2::GraphStorageServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::storage::cpp2::GraphStorageServiceAsyncProcessor*, apache::thrift::Tile*)::{lambda(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>)#1}::operator()(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>) (this=0x7f5e827c2040, rq=...)
    at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/async/AsyncProcessor.h:698
#11 0x00000000034fda90 in folly::detail::function::FunctionTraits<void (std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>)>::callBig<apache::thrift::GeneratedAsyncProcessor::makeEventTaskForRequest<nebula::storage::cpp2::GraphStorageServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::storage::cpp2::GraphStorageServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::storage::cpp2::GraphStorageServiceAsyncProcessor*, apache::thrift::Tile*)::{lambda(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>)#1}>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>&&, folly::detail::function::Data&) (args#0=..., p=...) at /root/src/nebula/build/third-party/install/include/folly/Function.h:385
#12 0x0000000004557c62 in apache::thrift::EventTask::run() ()
#13 0x00000000034acf63 in apache::thrift::GeneratedAsyncProcessor::processInThread<nebula::storage::cpp2::GraphStorageServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::storage::cpp2::GraphStorageServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::storage::cpp2::GraphStorageServiceAsyncProcessor*)::{lambda()#1}::operator()() const (this=0x7f5e865d7f90)
    at /root/src/nebula/build/third-party/install/include/thrift/lib/cpp2/async/AsyncProcessor.h:773
#14 0x00000000034e3d1e in folly::detail::function::FunctionTraits<void ()>::callSmall<apache::thrift::GeneratedAsyncProcessor::processInThread<nebula::storage::cpp2::GraphStorageServiceAsyncProcessor>(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*, apache::thrift::RpcKind, void (nebula::storage::cpp2::GraphStorageServiceAsyncProcessor::*)(std::unique_ptr<apache::thrift::ResponseChannelRequest, apache::thrift::RequestsRegistry::Deleter>, apache::thrift::SerializedRequest&&, apache::thrift::Cpp2RequestContext*, folly::EventBase*, apache::thrift::concurrency::ThreadManager*), nebula::storage::cpp2::GraphStorageServiceAsyncProcessor*)::{lambda()#1}>(folly::detail::function::Data&) (p=...) at /root/src/nebula/build/third-party/install/include/folly/Function.h:371
#15 0x000000000455d107 in virtual thunk to apache::thrift::concurrency::FunctionRunner::run() ()
#16 0x0000000004692503 in apache::thrift::concurrency::ThreadManager::Impl::Worker::run() ()
#17 0x000000000469615d in apache::thrift::concurrency::PthreadThread::threadMain(void*) ()
#18 0x00007f5e86b70609 in start_thread (arg=<optimized out>) at pthread_create.c:477
#19 0x00007f5e86a97293 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
(rr)
```

现在问题是：为什么 store2 在 index 12125 处的日志对应的 term 是 83？这条日志怎么来？谁给 store2 发的这条日志？term 83 的 leader 是谁？翻一下 store2 的日志：

```txt

5397 I1126 17:10:06.281394    32 RaftPart.cpp:1625] [Port: 9780, Space: 1, Part: 1] The current role is Follower. Will follow the new leader "store5":9780 on term 82
 5398 I1126 17:10:06.281534    32 RaftPart.cpp:1459] [Port: 9780, Space: 1, Part: 1] Local is missing logs from id 12121. Need to catch up
 5399 I1126 17:10:06.706071    84 RaftPart.cpp:1625] [Port: 9780, Space: 1, Part: 1] The current role is Follower. Will follow the new leader "store4":9780 on term 83
 5400 I1126 17:10:06.925935    32 RaftPart.cpp:1042] [Port: 9780, Space: 1, Part: 1] Partition's role has changed to Follower during the election, so discard the results
 5401 I1126 17:10:06.974997    32 RaftPart.cpp:1254] [Port: 9780, Space: 1, Part: 1] Received a VOTING request: space = 1, partition = 1, candidateAddr = store1:9780, term = 81, lastLogId = 12121, lastLogTerm = 80, isPreVote = 0
 5402 I1126 17:10:06.975021    32 RaftPart.cpp:1282] [Port: 9780, Space: 1, Part: 1] The partition currently is a Follower, lastLogId 12121, lastLogTerm 80, committedLogId 12120, term 83
 5403 I1126 17:10:06.975031    32 RaftPart.cpp:1299] [Port: 9780, Space: 1, Part: 1] The partition currently is on term 83, the term proposed by the candidate is 81, so it will be rejected
 5404 I1126 17:10:06.987138    32 RaftPart.cpp:1254] [Port: 9780, Space: 1, Part: 1] Received a VOTING request: space = 1, partition = 1, candidateAddr = store1:9780, term = 81, lastLogId = 12121, lastLogTerm = 80, isPreVote = 0
 5405 I1126 17:10:06.987159    32 RaftPart.cpp:1282] [Port: 9780, Space: 1, Part: 1] The partition currently is a Follower, lastLogId 12121, lastLogTerm 80, committedLogId 12120, term 83
 5406 I1126 17:10:06.987169    32 RaftPart.cpp:1299] [Port: 9780, Space: 1, Part: 1] The partition currently is on term 83, the term proposed by the candidate is 81, so it will be rejected
 5407 I1126 17:10:07.023480    32 RaftPart.cpp:1254] [Port: 9780, Space: 1, Part: 1] Received a VOTING request: space = 1, partition = 1, candidateAddr = store4:9780, term = 83, lastLogId = 12124, lastLogTerm = 82, isPreVote = 0
 5408 I1126 17:10:07.023502    32 RaftPart.cpp:1282] [Port: 9780, Space: 1, Part: 1] The partition currently is a Follower, lastLogId 12121, lastLogTerm 80, committedLogId 12120, term 83
 5409 I1126 17:10:07.023512    32 RaftPart.cpp:1355] [Port: 9780, Space: 1, Part: 1] The partition will vote for the candidate "store4":9780, isPreVote = 0
 5410 I1126 17:10:07.048154    32 RaftPart.cpp:1254] [Port: 9780, Space: 1, Part: 1] Received a VOTING request: space = 1, partition = 1, candidateAddr = store4:9780, term = 83, lastLogId = 12124, lastLogTerm = 82, isPreVote = 0
 5411 I1126 17:10:07.048177    32 RaftPart.cpp:1282] [Port: 9780, Space: 1, Part: 1] The partition currently is a Follower, lastLogId 12121, lastLogTerm 80, committedLogId 12120, term 83
 5412 I1126 17:10:07.048188    32 RaftPart.cpp:1355] [Port: 9780, Space: 1, Part: 1] The partition will vote for the candidate "store4":9780, isPreVote = 0
 5413 I1126 17:10:07.060901    32 RaftPart.cpp:1625] [Port: 9780, Space: 1, Part: 1] The current role is Follower. Will follow the new leader "store4":9780 on term 83
 5414 I1126 17:10:07.061043    32 RaftPart.cpp:1459] [Port: 9780, Space: 1, Part: 1] Local is missing logs from id 12121. Need to catch up
 5415 I1126 17:10:07.061187    32 RaftPart.cpp:1459] [Port: 9780, Space: 1, Part: 1] Local is missing logs from id 12121. Need to catch up
 5416 I1126 17:10:07.141144    79 Part.cpp:209] [Port: 9780, Space: 1, Part: 1] Find the new leader "store4":9780
 5417 I1126 17:10:07.141475    80 Part.cpp:209] [Port: 9780, Space: 1, Part: 1] Find the new leader "store4":9780
 5418 I1126 17:10:07.141769    81 Part.cpp:209] [Port: 9780, Space: 1, Part: 1] Find the new leader "store4":9780
 5419 I1126 17:10:07.234535    77 MetaClient.cpp:3013] Load leader of "store1":9779 in 0 space
 5420 I1126 17:10:07.234566    77 MetaClient.cpp:3013] Load leader of "store2":9779 in 0 space
 5421 I1126 17:10:07.234575    77 MetaClient.cpp:3013] Load leader of "store3":9779 in 0 space
 5422 I1126 17:10:07.234598    77 MetaClient.cpp:3013] Load leader of "store4":9779 in 1 space
 5423 I1126 17:10:07.234607    77 MetaClient.cpp:3013] Load leader of "store5":9779 in 0 space
 5424 I1126 17:10:07.234616    77 MetaClient.cpp:3019] Load leader ok
 5425 I1126 17:10:08.190805    32 RaftPart.cpp:1254] [Port: 9780, Space: 1, Part: 1] Received a VOTING request: space = 1, partition = 1, candidateAddr = store1:9780, term = 84, lastLogId = 12122, lastLogTerm = 81, isPreVote = 0
 5426 I1126 17:10:08.190836    32 RaftPart.cpp:1282] [Port: 9780, Space: 1, Part: 1] The partition currently is a Follower, lastLogId 12125, lastLogTerm 83, committedLogId 12123, term 83
 5427 I1126 17:10:08.190848    32 RaftPart.cpp:1314] [Port: 9780, Space: 1, Part: 1] The partition's last term to receive a log is 83, which is newer than the candidate's log 81. So the candidate will be rejected
 5428 I1126 17:10:08.262166    77 MetaClient.cpp:3013] Load leader of "store1":9779 in 0 space
 5429 I1126 17:10:08.262193    77 MetaClient.cpp:3013] Load leader of "store2":9779 in 0 space
 5430 I1126 17:10:08.262202    77 MetaClient.cpp:3013] Load leader of "store3":9779 in 0 space
 5431 I1126 17:10:08.262223    77 MetaClient.cpp:3013] Load leader of "store4":9779 in 1 space
 5432 I1126 17:10:08.262238    77 MetaClient.cpp:3013] Load leader of "store5":9779 in 0 space
 5433 I1126 17:10:08.262245    77 MetaClient.cpp:3019] Load leader ok
 5434 I1126 17:10:08.444456    63 RaftPart.cpp:1625] [Port: 9780, Space: 1, Part: 1] The current role is Follower. Will follow the new leader "store5":9780 on term 84
 5435 I1126 17:10:08.444599    63 RaftPart.cpp:1435] [Port: 9780, Space: 1, Part: 1] Stale log! The log 12122, term 81 i had committed yet. My committedLogId is 12123, term is 84
 5436 I1126 17:10:08.445242    79 Part.cpp:209] [Port: 9780, Space: 1, Part: 1] Find the new leader "store5":9780
 5437 I1126 17:10:09.289304    77 MetaClient.cpp:3013] Load leader of "store1":9779 in 0 space
 5438 I1126 17:10:09.289331    77 MetaClient.cpp:3013] Load leader of "store2":9779 in 0 space
 5439 I1126 17:10:09.289338    77 MetaClient.cpp:3013] Load leader of "store3":9779 in 0 space
 5440 I1126 17:10:09.289345    77 MetaClient.cpp:3013] Load leader of "store4":9779 in 0 space
 5441 I1126 17:10:09.289366    77 MetaClient.cpp:3013] Load leader of "store5":9779 in 1 space
 5442 I1126 17:10:09.289372    77 MetaClient.cpp:3019] Load leader ok
 5443 I1126 17:10:10.231608    63 RaftPart.cpp:1254] [Port: 9780, Space: 1, Part: 1] Received a VOTING request: space = 1, partition = 1, candidateAddr = store1:9780, term = 85, lastLogId = 12311, lastLogTerm = 84, isPreVote = 0
 5444 I1126 17:10:10.231637    63 RaftPart.cpp:1282] [Port: 9780, Space: 1, Part: 1] The partition currently is a Follower, lastLogId 12368, lastLogTerm 84, committedLogId 12311, term 84
 5445 I1126 17:10:10.231660    63 RaftPart.cpp:1324] [Port: 9780, Space: 1, Part: 1] The partition's last log id is 12368. The candidate's last log id 12311 is smaller, so it will be rejected, candidate is "store1":9780
 5446 I1126 17:10:10.748667    63 RaftPart.cpp:1254] [Port: 9780, Space: 1, Part: 1] Received a VOTING request: space = 1, partition = 1, candidateAddr = store3:9780, term = 86, lastLogId = 12368, lastLogTerm = 84, isPreVote = 0
 5447 I1126 17:10:10.748693    63 RaftPart.cpp:1282] [Port: 9780, Space: 1, Part: 1] The partition currently is a Follower, lastLogId 12368, lastLogTerm 84, committedLogId 12311, term 85
 5448 I1126 17:10:10.748713    63 RaftPart.cpp:1355] [Port: 9780, Space: 1, Part: 1] The partition will vote for the candidate "store3":9780, isPreVote = 0
```

![storage-raft-logs1](/images/raft-consist/log1.jpg)

可以看到 term 83 对应的 leader 是 store4。日志里还有一行：Local is missing logs from id 12121. Need to catch up. 把打印日志的这段代码 RaftPart.cpp:1459 抽出来看一下：

```cpp
void RaftPart::processAppendLogRequest(const cpp2::AppendLogRequest& req,
                                       cpp2::AppendLogResponse& resp) {
  LOG_IF(INFO, FLAGS_trace_raft) << idStr_ << "Received logAppend"
                                 << ": GraphSpaceId = " << req.get_space()
                                 << ", partition = " << req.get_part()
                                 << ", leaderIp = " << req.get_leader_addr()
                                 << ", leaderPort = " << req.get_leader_port()
                                 << ", current_term = " << req.get_current_term()
                                 << ", lastLogId = " << req.get_last_log_id()
                                 << ", committedLogId = " << req.get_committed_log_id()
                                 << ", lastLogIdSent = " << req.get_last_log_id_sent()
                                 << ", lastLogTermSent = " << req.get_last_log_term_sent()
                                 << ", num_logs = " << req.get_log_str_list().size()
                                 << ", logTerm = " << req.get_log_term()
                                 << ", local lastLogId = " << lastLogId_
                                 << ", local lastLogTerm = " << lastLogTerm_
                                 << ", local committedLogId = " << committedLogId_
                                 << ", local current term = " << term_
                                 << ", wal lastLogId = " << wal_->lastLogId();
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
  if (UNLIKELY(status_ == Status::STARTING)) {
    VLOG(2) << idStr_ << "The partition is still starting";
    resp.set_error_code(cpp2::ErrorCode::E_NOT_READY);
    return;
  }
  if (UNLIKELY(status_ == Status::WAITING_SNAPSHOT)) {
    VLOG(2) << idStr_ << "The partition is waiting for snapshot";
    resp.set_error_code(cpp2::ErrorCode::E_WAITING_SNAPSHOT);
    return;
  }
  // Check leadership
  cpp2::ErrorCode err = verifyLeader<cpp2::AppendLogRequest>(req);
  if (err != cpp2::ErrorCode::SUCCEEDED) {
    // Wrong leadership
    VLOG(2) << idStr_ << "Will not follow the leader";
    resp.set_error_code(err);
    return;
  }
 
  // Reset the timeout timer
  lastMsgRecvDur_.reset();
 
  if (req.get_last_log_id_sent() < committedLogId_ && req.get_last_log_term_sent() <= term_) {
    LOG(INFO) << idStr_ << "Stale log! The log " << req.get_last_log_id_sent() << ", term "
              << req.get_last_log_term_sent() << " i had committed yet. My committedLogId is "
              << committedLogId_ << ", term is " << term_;
    resp.set_error_code(cpp2::ErrorCode::E_LOG_STALE);
    return;
  } else if (req.get_last_log_id_sent() < committedLogId_) {
    LOG(INFO) << idStr_ << "What?? How it happens! The log id is " << req.get_last_log_id_sent()
              << ", the log term is " << req.get_last_log_term_sent()
              << ", but my committedLogId is " << committedLogId_ << ", my term is " << term_
              << ", to make the cluster stable i will follow the high term"
              << " candidate and cleanup my data";
    reset();
    resp.set_committed_log_id(committedLogId_);
    resp.set_last_log_id(lastLogId_);
    resp.set_last_log_term(lastLogTerm_);
    return;
  }
 
  // req.get_last_log_id_sent() >= committedLogId_
  if (req.get_last_log_id_sent() == lastLogId_ && req.get_last_log_term_sent() == lastLogTerm_) {
    // nothing to do
    // just append log later
  } else if (req.get_last_log_id_sent() > lastLogId_) {
    // There is a gap
    LOG(INFO) << idStr_ << "Local is missing logs from id " << lastLogId_ << ". Need to catch up";
    resp.set_error_code(cpp2::ErrorCode::E_LOG_GAP);
    return;
  } else {
    // check the last log term is matched or not
    int reqLastLogTerm = wal_->getLogTerm(req.get_last_log_id_sent());
    if (req.get_last_log_term_sent() != reqLastLogTerm) {
      LOG(INFO) << idStr_ << "The local log term is " << reqLastLogTerm
                << ", which is different from the leader's prevLogTerm "
                << req.get_last_log_term_sent() << ", the prevLogId is "
                << req.get_last_log_id_sent() << ". So ask leader to send logs from committedLogId "
                << committedLogId_;
      TermID committedLogTerm = wal_->getLogTerm(committedLogId_);
      if (committedLogTerm > 0) {
        resp.set_last_log_id(committedLogId_);
        resp.set_last_log_term(committedLogTerm);
      }
      resp.set_error_code(cpp2::ErrorCode::E_LOG_GAP);
      return;
    }
  }
```

通过 rr 回放 store2 的执行过程，并且在 RaftPart.cpp:1459 下断点，循环几次后我们准确定位到 index = 12121 时在 RaftPart.cpp:1459 这行打印的日志：

```txt
Thread 10 hit Breakpoint 1, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1459
1459        LOG(INFO) << idStr_ << "Local is missing logs from id " << lastLogId_ << ". Need to catch up";
1: lastLogId_ = 10439
(rr)
Continuing.
[Switching to Thread 25.63]
 
Thread 40 hit Breakpoint 1, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1459
1459        LOG(INFO) << idStr_ << "Local is missing logs from id " << lastLogId_ << ". Need to catch up";
1: lastLogId_ = 10446
(rr)
Continuing.
 
Thread 40 hit Breakpoint 1, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1459
1459        LOG(INFO) << idStr_ << "Local is missing logs from id " << lastLogId_ << ". Need to catch up";
1: lastLogId_ = 10970
(rr)
Continuing.
[Switching to Thread 25.34]
 
Thread 11 hit Breakpoint 1, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1459
1459        LOG(INFO) << idStr_ << "Local is missing logs from id " << lastLogId_ << ". Need to catch up";
1: lastLogId_ = 11143
(rr)
Continuing.
[Switching to Thread 25.32]
 
Thread 3 hit Breakpoint 1, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1459
1459        LOG(INFO) << idStr_ << "Local is missing logs from id " << lastLogId_ << ". Need to catch up";
1: lastLogId_ = 12121
(rr)
```

看下当时发这个 appendLog 请求的 leader 是谁：

```txt
(rr) p req.get_leader_addr()
$8 = (const std::string &) @0x7f7ff38d1870: {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f7ff38d1880 "store5"}, _M_string_length = 6, {
    _M_local_buf = "store5\000\000 \031\215\363\177\177\000", _M_allocated_capacity = 58709827875955}}
(rr)
```

竟然是 store5！store2 此时的 term 呢？

```txt
(rr) p term_
$9 = 82
```

store2 的 term 是 82……难道 12121 这个要多次踩点？看日志里面，有三行，好吧，继续 continue：

![raft-storage-logs2](/images/raft-consist/log2.jpg)

在 continue 几次，定位到 term = 83 时这条日志打印的执行点：

```txt
Thread 3 hit Breakpoint 1, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1459
1459        LOG(INFO) << idStr_ << "Local is missing logs from id " << lastLogId_ << ". Need to catch up";
1: lastLogId_ = 12420
(rr) reverse-cont
Continuing.
 
Thread 3 hit Breakpoint 1, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1459
1459        LOG(INFO) << idStr_ << "Local is missing logs from id " << lastLogId_ << ". Need to catch up";
1: lastLogId_ = 12121
(rr) p req.get_leader_addr()
$13 = (const std::string &) @0x7f7ff38d1870: {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f7ff38d1880 "store4"}, _M_string_length = 6, {
    _M_local_buf = "store4\000\000 \031\215\363\177\177\000", _M_allocated_capacity = 57610316248179}}
(rr) p term_
$14 = 83
```

这时候的 leader 居然是 store4！哦，不过没啥惊讶的，和 store2 的日志对的上。回到 store2 的日志上：

![raft-storage-logs3](/images/raft-consist/log3.jpg)

打印这行日志的代码 RaftPart.cpp:1282：

```txt

void RaftPart::processAskForVoteRequest(const cpp2::AskForVoteRequest& req,
                                        cpp2::AskForVoteResponse& resp) {
  LOG(INFO) << idStr_ << "Received a VOTING request"
            << ": space = " << req.get_space() << ", partition = " << req.get_part()
            << ", candidateAddr = " << req.get_candidate_addr() << ":" << req.get_candidate_port()
            << ", term = " << req.get_term() << ", lastLogId = " << req.get_last_log_id()
            << ", lastLogTerm = " << req.get_last_log_term()
            << ", isPreVote = " << req.get_is_pre_vote();
 
  std::lock_guard<std::mutex> g(raftLock_);
 
  // Make sure the partition is running
  if (UNLIKELY(status_ == Status::STOPPED)) {
    LOG(INFO) << idStr_ << "The part has been stopped, skip the request";
    resp.set_error_code(cpp2::ErrorCode::E_BAD_STATE);
    return;
  }
 
  if (UNLIKELY(status_ == Status::STARTING)) {
    LOG(INFO) << idStr_ << "The partition is still starting";
    resp.set_error_code(cpp2::ErrorCode::E_NOT_READY);
    return;
  }
 
  if (UNLIKELY(status_ == Status::WAITING_SNAPSHOT)) {
    LOG(INFO) << idStr_ << "The partition is still waiting snapshot";
    resp.set_error_code(cpp2::ErrorCode::E_WAITING_SNAPSHOT);
    return;
  }
 
  LOG(INFO) << idStr_ << "The partition currently is a " << roleStr(role_) << ", lastLogId "
            << lastLogId_ << ", lastLogTerm " << lastLogTerm_ << ", committedLogId "
            << committedLogId_ << ", term " << term_;
  if (role_ == Role::LEARNER) {
    resp.set_error_code(cpp2::ErrorCode::E_BAD_ROLE);
    return;
  }
```

store2 的 lastLogIndex 什么时候悄咪咪变成 12125 了 —— 也就是 store2 出问题的那条 log index。
ok，接下来我们要分析的就是 12125 这条日志时上面时候进来的，
又是怎么 commit 上去（一个猜测不一定对：
store4 把错误的 12125 日志带进来，
store5 成为 term 84 的 leader 后，
Raft 没把这条 log revert 导致它被错误的 commit 上去）。
我们接下来在 RaftPart:processAppendLogRequest() 上下断点已观察 12121 - 12125 这个 index 范围内 store2 的 raft append log 操作。

```txt
Thread 3 hit Breakpoint 3, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1434
1434      if (req.get_last_log_id_sent() < committedLogId_ && req.get_last_log_term_sent() <= term_) {
1: lastLogId_ = 12121
(rr) p req
$34 = (const nebula::raftex::cpp2::AppendLogRequest &) @0x7f7ff38d1850: {static __fbthrift_cpp2_gen_json = false, static __fbthrift_cpp2_gen_nimble = false, static __fbthrift_cpp2_gen_has_thrift_uri = false, static __fbthrift_cpp2_is_union = false, space = 1, part = 1,
  current_term = 83, last_log_id = 12125, committed_log_id = 12123, leader_addr = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f7ff38d1880 "store4"},
    _M_string_length = 6, {_M_local_buf = "store4\000\000 \031\215\363\177\177\000", _M_allocated_capacity = 57610316248179}}, leader_port = 9780, last_log_term_sent = 80, last_log_id_sent = 12121, log_term = 81,
  log_str_list = {<std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >> = {
      _M_impl = {<std::allocator<nebula::cpp2::LogEntry>> = {<__gnu_cxx::new_allocator<nebula::cpp2::LogEntry>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >::_Vector_impl_data> = {
          _M_start = 0x7f7fe3a4a7c0, _M_finish = 0x7f7fe3a4a7f0, _M_end_of_storage = 0x7f7fe3a4a7f0}, <No data fields>}}, <No data fields>}, __isset = {space = true, part = true, current_term = true, last_log_id = true, committed_log_id = true, leader_addr = true,
    leader_port = true, last_log_term_sent = true, last_log_id_sent = true, log_term = true, log_str_list = true}}
(rr) p req.get_current_term()
$35 = 83
(rr) p req.get_log_str_list().size()
$36 = 1
```

store4 给 store2 发了一条长度 1 的 log，继续跟。

```txt
(rr) c
Continuing.
 
Thread 3 hit Breakpoint 3, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1434
1434      if (req.get_last_log_id_sent() < committedLogId_ && req.get_last_log_term_sent() <= term_) {
1: lastLogId_ = 12122
(rr) p req
$37 = (const nebula::raftex::cpp2::AppendLogRequest &) @0x7f7ff38d1850: {static __fbthrift_cpp2_gen_json = false, static __fbthrift_cpp2_gen_nimble = false, static __fbthrift_cpp2_gen_has_thrift_uri = false, static __fbthrift_cpp2_is_union = false, space = 1, part = 1,
  current_term = 83, last_log_id = 12125, committed_log_id = 12123, leader_addr = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f7ff38d1880 "store4"},
    _M_string_length = 6, {_M_local_buf = "store4\000\000 \031\215\363\177\177\000", _M_allocated_capacity = 57610316248179}}, leader_port = 9780, last_log_term_sent = 81, last_log_id_sent = 12122, log_term = 82,
  log_str_list = {<std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >> = {
      _M_impl = {<std::allocator<nebula::cpp2::LogEntry>> = {<__gnu_cxx::new_allocator<nebula::cpp2::LogEntry>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >::_Vector_impl_data> = {
          _M_start = 0x7f7fbf2b3f00, _M_finish = 0x7f7fbf2b3f60, _M_end_of_storage = 0x7f7fbf2b3f60}, <No data fields>}}, <No data fields>}, __isset = {space = true, part = true, current_term = true, last_log_id = true, committed_log_id = true, leader_addr = true,
    leader_port = true, last_log_term_sent = true, last_log_id_sent = true, log_term = true, log_str_list = true}}
(rr)
```

连续 continue 几次，我们来到了 store2 lastLogIndex = 12124 这个关键点：

```txt
(rr) c
Continuing.
 
Thread 3 hit Breakpoint 3, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1434
1434      if (req.get_last_log_id_sent() < committedLogId_ && req.get_last_log_term_sent() <= term_) {
1: lastLogId_ = 12124
2: req = (const nebula::raftex::cpp2::AppendLogRequest &) @0x7f7ff38d1850: {static __fbthrift_cpp2_gen_json = false, static __fbthrift_cpp2_gen_nimble = false, static __fbthrift_cpp2_gen_has_thrift_uri = false, static __fbthrift_cpp2_is_union = false, space = 1, part = 1,
  current_term = 83, last_log_id = 12125, committed_log_id = 12123, leader_addr = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f7ff38d1880 "store4"},
    _M_string_length = 6, {_M_local_buf = "store4\000\000 \031\215\363\177\177\000", _M_allocated_capacity = 57610316248179}}, leader_port = 9780, last_log_term_sent = 82, last_log_id_sent = 12124, log_term = 83,
  log_str_list = {<std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >> = {
      _M_impl = {<std::allocator<nebula::cpp2::LogEntry>> = {<__gnu_cxx::new_allocator<nebula::cpp2::LogEntry>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >::_Vector_impl_data> = {
          _M_start = 0x7f7fe3a4a7c0, _M_finish = 0x7f7fe3a4a7f0, _M_end_of_storage = 0x7f7fe3a4a7f0}, <No data fields>}}, <No data fields>}, __isset = {space = true, part = true, current_term = true, last_log_id = true, committed_log_id = true, leader_addr = true,
    leader_port = true, last_log_term_sent = true, last_log_id_sent = true, log_term = true, log_str_list = true}}
(rr)
```

此时 leader store4 上的信息：

1. commitedLogIndex = 12123
2. req.get_log_str_list().size() = 1—— appendLog 请求中 log 列表长度

```txt

(rr) n
1483      LogID firstId = req.get_last_log_id_sent() + 1;
1: lastLogId_ = 12124
2: req = (const nebula::raftex::cpp2::AppendLogRequest &) @0x7f7ff38d1850: {static __fbthrift_cpp2_gen_json = false, static __fbthrift_cpp2_gen_nimble = false, static __fbthrift_cpp2_gen_has_thrift_uri = false, static __fbthrift_cpp2_is_union = false, space = 1, part = 1,
  current_term = 83, last_log_id = 12125, committed_log_id = 12123, leader_addr = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f7ff38d1880 "store4"},
    _M_string_length = 6, {_M_local_buf = "store4\000\000 \031\215\363\177\177\000", _M_allocated_capacity = 57610316248179}}, leader_port = 9780, last_log_term_sent = 82, last_log_id_sent = 12124, log_term = 83,
  log_str_list = {<std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >> = {
      _M_impl = {<std::allocator<nebula::cpp2::LogEntry>> = {<__gnu_cxx::new_allocator<nebula::cpp2::LogEntry>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >::_Vector_impl_data> = {
          _M_start = 0x7f7fe3a4a7c0, _M_finish = 0x7f7fe3a4a7f0, _M_end_of_storage = 0x7f7fe3a4a7f0}, <No data fields>}}, <No data fields>}, __isset = {space = true, part = true, current_term = true, last_log_id = true, committed_log_id = true, leader_addr = true,
    leader_port = true, last_log_term_sent = true, last_log_id_sent = true, log_term = true, log_str_list = true}}
(rr) n
1485      size_t diffIndex = 0;
1: lastLogId_ = 12124
2: req = (const nebula::raftex::cpp2::AppendLogRequest &) @0x7f7ff38d1850: {static __fbthrift_cpp2_gen_json = false, static __fbthrift_cpp2_gen_nimble = false, static __fbthrift_cpp2_gen_has_thrift_uri = false, static __fbthrift_cpp2_is_union = false, space = 1, part = 1,
  current_term = 83, last_log_id = 12125, committed_log_id = 12123, leader_addr = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f7ff38d1880 "store4"},
    _M_string_length = 6, {_M_local_buf = "store4\000\000 \031\215\363\177\177\000", _M_allocated_capacity = 57610316248179}}, leader_port = 9780, last_log_term_sent = 82, last_log_id_sent = 12124, log_term = 83,
  log_str_list = {<std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >> = {
      _M_impl = {<std::allocator<nebula::cpp2::LogEntry>> = {<__gnu_cxx::new_allocator<nebula::cpp2::LogEntry>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >::_Vector_impl_data> = {
          _M_start = 0x7f7fe3a4a7c0, _M_finish = 0x7f7fe3a4a7f0, _M_end_of_storage = 0x7f7fe3a4a7f0}, <No data fields>}}, <No data fields>}, __isset = {space = true, part = true, current_term = true, last_log_id = true, committed_log_id = true, leader_addr = true,
    leader_port = true, last_log_term_sent = true, last_log_id_sent = true, log_term = true, log_str_list = true}}
(rr) p firstId
$58 = 12125
(rr)
```

终于定位到 12125 这条 log 了！
我看到了 12125 这条日志插入的全过程，
因为 leader 的 commitedLogIndex = 12123，
所以他并没有被 commit。所以现在的问题时，
12125 这条本应该被 rollback 的 log 为什么被 commit？
还是通过原来 appendLogs 的断点来跟踪。
又经过几次 continue，我们来到了一个激动人心的地方：

```txt
Continuing.
 
Thread 40 hit Breakpoint 3, nebula::raftex::RaftPart::processAppendLogRequest (this=0x7f7fbf100010, req=..., resp=...) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:1434
1434      if (req.get_last_log_id_sent() < committedLogId_ && req.get_last_log_term_sent() <= term_) {
1: lastLogId_ = 12125
2: req = (const nebula::raftex::cpp2::AppendLogRequest &) @0x7f7feb932850: {static __fbthrift_cpp2_gen_json = false, static __fbthrift_cpp2_gen_nimble = false, static __fbthrift_cpp2_gen_has_thrift_uri = false, static __fbthrift_cpp2_is_union = false, space = 1, part = 1,
  current_term = 84, last_log_id = 12368, committed_log_id = 12311, leader_addr = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f7feb932880 "store5"},
    _M_string_length = 6, {_M_local_buf = "store5\000\000 )\223\353\177\177\000", _M_allocated_capacity = 58709827875955}}, leader_port = 9780, last_log_term_sent = 83, last_log_id_sent = 12125, log_term = 84,
  log_str_list = {<std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >> = {
      _M_impl = {<std::allocator<nebula::cpp2::LogEntry>> = {<__gnu_cxx::new_allocator<nebula::cpp2::LogEntry>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >::_Vector_impl_data> = {
          _M_start = 0x7f7fe38c1800, _M_finish = 0x7f7fe38c3000, _M_end_of_storage = 0x7f7fe38c3000}, <No data fields>}}, <No data fields>}, __isset = {space = true, part = true, current_term = true, last_log_id = true, committed_log_id = true, leader_addr = true,
    leader_port = true, last_log_term_sent = true, last_log_id_sent = true, log_term = true, log_str_list = true}}
3: committedLogId_ = 12123
(rr)
```

此时：

1. store2 的 commitedLogIndex = 12123
2. store2 的 lasteLogId = 12125
3. leader 是 store5
4. store5 的 commitedLogIndex = 12311
5. store5 发过来的请求的 last_log_id_sent = 12125（这个命名有点绕，应该就是对应的 Raft 中的 prevLogIndex 来理解）
6. store5 发过来的请求的 last_log_term_sent = 83 （如果这表示的的是 index 12125 的 term 那问题就严重了，store5 上 index 12125 的 term 是 84 而不是 83）

按照预期，store5 应该吧 store2 index = 12125 的这条日志 rollback，我们接下来就看看为什么没有 rollback，反而吧它 commit 了。继续跟踪：

```txt
(rr) n
1454      if (req.get_last_log_id_sent() == lastLogId_ && req.get_last_log_term_sent() == lastLogTerm_) {
1: lastLogId_ = 12125
2: req = (const nebula::raftex::cpp2::AppendLogRequest &) @0x7f7feb932850: {static __fbthrift_cpp2_gen_json = false, static __fbthrift_cpp2_gen_nimble = false, static __fbthrift_cpp2_gen_has_thrift_uri = false, static __fbthrift_cpp2_is_union = false, space = 1, part = 1,
  current_term = 84, last_log_id = 12368, committed_log_id = 12311, leader_addr = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f7feb932880 "store5"},
    _M_string_length = 6, {_M_local_buf = "store5\000\000 )\223\353\177\177\000", _M_allocated_capacity = 58709827875955}}, leader_port = 9780, last_log_term_sent = 83, last_log_id_sent = 12125, log_term = 84,
  log_str_list = {<std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >> = {
      _M_impl = {<std::allocator<nebula::cpp2::LogEntry>> = {<__gnu_cxx::new_allocator<nebula::cpp2::LogEntry>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >::_Vector_impl_data> = {
          _M_start = 0x7f7fe38c1800, _M_finish = 0x7f7fe38c3000, _M_end_of_storage = 0x7f7fe38c3000}, <No data fields>}}, <No data fields>}, __isset = {space = true, part = true, current_term = true, last_log_id = true, committed_log_id = true, leader_addr = true,
    leader_port = true, last_log_term_sent = true, last_log_id_sent = true, log_term = true, log_str_list = true}}
3: committedLogId_ = 12123
(rr) p req.get_last_log_id_sent()
$67 = 12125
(rr) p req.get_last_log_term_sent()
$68 = 83
(rr) p lastLogId_
$69 = 12125
(rr) p lastLogTerm_
$70 = 83
(rr)
```

啊，我们几乎找到问题的直接原因了？store5 发过来的 log 信息是错的，查代码验证上面关于 log 字段的用法：

```cpp
// req.get_last_log_id_sent() >= committedLogId_
if (req.get_last_log_id_sent() == lastLogId_ && req.get_last_log_term_sent() == lastLogTerm_) {
  // nothing to do
  // just append log later
} else if (req.get_last_log_id_sent() > lastLogId_) {
  // There is a gap
  LOG(INFO) << idStr_ << "Local is missing logs from id " << lastLogId_ << ". Need to catch up";
  resp.set_error_code(cpp2::ErrorCode::E_LOG_GAP);
  return;
} else {
  // check the last log term is matched or not
  int reqLastLogTerm = wal_->getLogTerm(req.get_last_log_id_sent());
  if (req.get_last_log_term_sent() != reqLastLogTerm) {
    LOG(INFO) << idStr_ << "The local log term is " << reqLastLogTerm
```

现在得去看 store5 为啥抽风发错误的信息。我们查了一遍 store5 的 raft wal log 记录：

```txt
12115 log index: 12114, term: 75, logsz: 0, cluster_id: 0, walfile:
12116 log index: 12115, term: 78, logsz: 0, cluster_id: 0, walfile:
12117 log index: 12116, term: 78, logsz: 0, cluster_id: 0, walfile:
12118 log index: 12117, term: 78, logsz: 0, cluster_id: 0, walfile:
12119 log index: 12118, term: 79, logsz: 0, cluster_id: 0, walfile:
12120 log index: 12119, term: 79, logsz: 0, cluster_id: 0, walfile:
12121 log index: 12120, term: 79, logsz: 0, cluster_id: 0, walfile:
12122 log index: 12121, term: 80, logsz: 0, cluster_id: 0, walfile:
12123 log index: 12122, term: 81, logsz: 0, cluster_id: 0, walfile:
12124 log index: 12123, term: 82, logsz: 0, cluster_id: 0, walfile:
12125 log index: 12124, term: 82, logsz: 0, cluster_id: 0, walfile:
12126 log index: 12125, term: 84, logsz: 53, cluster_id: 0, walfile:
12127 log index: 12126, term: 84, logsz: 53, cluster_id: 0, walfile:
12128 log index: 12127, term: 84, logsz: 51, cluster_id: 0, walfile:
12129 log index: 12128, term: 84, logsz: 53, cluster_id: 0, walfile:
12130 log index: 12129, term: 84, logsz: 53, cluster_id: 0, walfile:
12131 log index: 12130, term: 84, logsz: 53, cluster_id: 0, walfile:
12132 log index: 12131, term: 84, logsz: 53, cluster_id: 0, walfile:
12133 log index: 12132, term: 84, logsz: 53, cluster_id: 0, walfile:
```

诡异的是 store5 这货根本没有 term = 83 的 log。
我们现在切到 store5 的执行现场，
在 RaftPart::replicateLogs() 这个函数上下断点，
运气不错，一下子就定位到 index 12125 的 log 位置：

```txt
(rr) c
Continuing.
 
Thread 5 hit Breakpoint 5, nebula::raftex::RaftPart::replicateLogs (this=0x7f5e4ed00010, eb=0x7f5e79b96f00, iter=..., currTerm=84, lastLogId=12125, committedId=12124, prevLogTerm=82, prevLogId=12124) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:753
753       AppendLogResult res = AppendLogResult::SUCCEEDED;
(rr) p iter.firstLogId()
$2 = 12125
(rr)
```

iter.firstLogId() == 12125，
term 是多少呢？
append log 的实际断点在 Host.cpp:appendLogs()中，
我们在在 Host.cpp:141 处下个断点，几次 continue 后，逮到大鱼了：

```txt

(rr) c
Continuing.
[Switching to Thread 25.71]
 
Thread 47 hit Breakpoint 8, nebula::raftex::Host::appendLogs (this=0x7f5e4ea3ad10, eb=0x7f5e79c32f00, term=84, logId=12186, committedLogId=12125, prevLogTerm=84, prevLogId=12125) at /root/src/nebula/src/kvstore/raftex/Host.cpp:141
141       appendLogsInternal(eb, std::move(req));
(rr) p idStr_
$24 = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f5e7357a100 "[Port: 9780, Space: 1, Part: 1] [Host: store2:9780] "}, _M_string_length = 52, {
    _M_local_buf = "4", '\000' <repeats 14 times>, _M_allocated_capacity = 52}}
(rr) p *req
$25 = (std::__shared_ptr_access<nebula::raftex::cpp2::AppendLogRequest, (__gnu_cxx::_Lock_policy)2, false, false>::element_type &) @0x7f5e79cb1130: {static __fbthrift_cpp2_gen_json = false, static __fbthrift_cpp2_gen_nimble = false,
  static __fbthrift_cpp2_gen_has_thrift_uri = false, static __fbthrift_cpp2_is_union = false, space = 1, part = 1, current_term = 84, last_log_id = 12186, committed_log_id = 12125, leader_addr = {static npos = 18446744073709551615,
    _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f5e79cb1160 "store5"}, _M_string_length = 6, {_M_local_buf = "store5\000\000\376\377\377\377\000\000\000", _M_allocated_capacity = 58709827875955}},
  leader_port = 9780, last_log_term_sent = 83, last_log_id_sent = 12125, log_term = 84, log_str_list = {<std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >> = {
      _M_impl = {<std::allocator<nebula::cpp2::LogEntry>> = {<__gnu_cxx::new_allocator<nebula::cpp2::LogEntry>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >::_Vector_impl_data> = {
          _M_start = 0x7f5e4ebd0800, _M_finish = 0x7f5e4ebd1370, _M_end_of_storage = 0x7f5e4ebd1400}, <No data fields>}}, <No data fields>}, __isset = {space = true, part = true, current_term = true, last_log_id = true, committed_log_id = true, leader_addr = true,
    leader_port = true, last_log_term_sent = true, last_log_id_sent = true, log_term = true, log_str_list = true}}
(rr) p req->get_last_log_term_sent()
$26 = 83
(rr) p req->get_last_log_id_sent()
$27 = 12125
(rr)
```

log index = 12125 && term = 83 的这条错误请求被我抓到了。
分析他的上下问确认为什么 term 被设置成 83 了。
我们通过代码回放，发现 store5 上 store2 的 Host 实例记录的信息：

1. lastLogIdSent_ = 12125
2. lastLogTermSent_ = 83
 
针对 lastLogIdSent_ 和 lastLogTermSent_ 下两个硬件断点，然后反相执行，定位到这两个字段上一次变更的位置和上下文：

```txt

(rr) watch lastLogIdSent_
Hardware watchpoint 10: lastLogIdSent_
(rr) watch lastLogTermSent_
Hardware watchpoint 11: lastLogTermSent_
(rr) reverse-cont
Continuing.
 
Thread 47 hit Breakpoint 9, nebula::raftex::Host::appendLogs (this=0x7f5e4ea3ad10, eb=0x7f5e79c32f00, term=84, logId=12186, committedLogId=12125, prevLogTerm=84, prevLogId=12125) at /root/src/nebula/src/kvstore/raftex/Host.cpp:81
81        std::shared_ptr<cpp2::AppendLogRequest> req;
1: *req = <error: Cannot call functions in reverse mode.>
(rr) p idStr_
$34 = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f5e7357a100 "[Port: 9780, Space: 1, Part: 1] [Host: store2:9780] "}, _M_string_length = 52, {
    _M_local_buf = "4", '\000' <repeats 14 times>, _M_allocated_capacity = 52}}
(rr) p lastLogTermSent_
$35 = 83
(rr) p lastLogIdSent_
$36 = 12125
(rr) reverse-cont
Continuing.
 
Thread 47 hit Hardware watchpoint 10: lastLogIdSent_
 
Old value = 12125
New value = 15
 
Thread 47 hit Hardware watchpoint 11: lastLogTermSent_
 
Old value = 83
New value = 8097318638240032884
0x00000000040c4603 in nebula::raftex::Host::appendLogs (this=0x5d88100, eb=0x0, term=140043699265904, logId=73728856, committedLogId=140043699266496, prevLogTerm=84, prevLogId=12125) at /root/src/nebula/src/kvstore/raftex/Host.cpp:77
77                                                              LogID prevLogId) {
1: *req = <error: Cannot call functions in reverse mode.>
(rr)
```

跑到这里不知道哪个指令出错，
rr 突然崩了……
重启 rr，换个策略，
给 committedLogId_ 下个硬件断点，
continue 几次，捕捉到 committedLogId_ 被设置为 83 的那个点：

```txt
(rr) c
Continuing.
[Switching to Thread 25.76]
 
Thread 52 hit Hardware watchpoint 2: *(int*) 0x7f5e4ea3afa0
 
Old value = 0
New value = 81
nebula::raftex::Host::appendLogs (this=0x7f5e4ea3ad10, eb=0x7f5e79e26f00, term=82, logId=12123, committedLogId=12119, prevLogTerm=81, prevLogId=12122) at /root/src/nebula/src/kvstore/raftex/Host.cpp:114
114           LOG(INFO) << idStr_ << "This is the first time to send the logs to this host"
(rr) c
Continuing.
[Switching to Thread 25.68]
 
Thread 44 hit Hardware watchpoint 2: *(int*) 0x7f5e4ea3afa0
 
Old value = 81
New value = 83
nebula::raftex::Host::<lambda(nebula::raftex::cpp2::AppendLogResponse&&)>::operator()(nebula::raftex::cpp2::AppendLogResponse &&) const (__closure=0x7f5e734cdc10, resp=...) at /root/src/nebula/src/kvstore/raftex/Host.cpp:187
187                   self->followerCommittedLogId_ = resp.get_committed_log_id();
(rr)
```

出问题的代码在 Host.cpp:186 行上：

```cpp
void Host::appendLogsInternal(folly::EventBase* eb, std::shared_ptr<cpp2::AppendLogRequest> req) {
  using TransportException = apache::thrift::transport::TTransportException;
  sendAppendLogRequest(eb, req)
      .via(eb)
      .thenValue([eb, self = shared_from_this()](cpp2::AppendLogResponse&& resp) {
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
...
```

我们在 appendLogsInternal() 上下了断点，然后通过 rr 反向执行定位到这个请求发起的时间点：

```txt
(rr) reverse-cont
Continuing.
 
Thread 44 hit Breakpoint 3, nebula::raftex::Host::appendLogsInternal (this=0x7f5e4ea3ad10, eb=0x7f5e79b96f00, req=...) at /root/src/nebula/src/kvstore/raftex/Host.cpp:158
158       sendAppendLogRequest(eb, req)
(rr) p idStr_
$13 = {static npos = 18446744073709551615, _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f5e7357a100 "[Port: 9780, Space: 1, Part: 1] [Host: store2:9780] "}, _M_string_length = 52, {
    _M_local_buf = "4", '\000' <repeats 14 times>, _M_allocated_capacity = 52}}
(rr) p lastLogTermSent_
$14 = 81
(rr) p req
$15 = {<std::__shared_ptr<nebula::raftex::cpp2::AppendLogRequest, (__gnu_cxx::_Lock_policy)2>> = {<std::__shared_ptr_access<nebula::raftex::cpp2::AppendLogRequest, (__gnu_cxx::_Lock_policy)2, false, false>> = {<No data fields>}, _M_ptr = 0x7f5e79bae0f0, _M_refcount = {
      _M_pi = 0x7f5e79bae0e0}}, <No data fields>}
(rr) p *req
$16 = (std::__shared_ptr_access<nebula::raftex::cpp2::AppendLogRequest, (__gnu_cxx::_Lock_policy)2, false, false>::element_type &) @0x7f5e79bae0f0: {static __fbthrift_cpp2_gen_json = false, static __fbthrift_cpp2_gen_nimble = false,
  static __fbthrift_cpp2_gen_has_thrift_uri = false, static __fbthrift_cpp2_is_union = false, space = 1, part = 1, current_term = 84, last_log_id = 12125, committed_log_id = 12124, leader_addr = {static npos = 18446744073709551615,
    _M_dataplus = {<std::allocator<char>> = {<__gnu_cxx::new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7f5e79bae120 "store5"}, _M_string_length = 6, {_M_local_buf = "store5\000\000\376\377\377\377\000\000\000", _M_allocated_capacity = 58709827875955}},
  leader_port = 9780, last_log_term_sent = 81, last_log_id_sent = 12122, log_term = 82, log_str_list = {<std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >> = {
      _M_impl = {<std::allocator<nebula::cpp2::LogEntry>> = {<__gnu_cxx::new_allocator<nebula::cpp2::LogEntry>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<nebula::cpp2::LogEntry, std::allocator<nebula::cpp2::LogEntry> >::_Vector_impl_data> = {
          _M_start = 0x7f5e79de2760, _M_finish = 0x7f5e79de27c0, _M_end_of_storage = 0x7f5e79de27c0}, <No data fields>}}, <No data fields>}, __isset = {space = true, part = true, current_term = true, last_log_id = true, committed_log_id = true, leader_addr = true,
    leader_port = true, last_log_term_sent = true, last_log_id_sent = true, log_term = true, log_str_list = true}}
(rr)
```

问题就出在这个请求上，
store5 给 store2 发了个请求，
store5 收到回应后根据 resp.get_last_log_term() 把 lastLogTermSent_ 设置成一个 store5 上不存在的 term 83。
现在的问题：

1. store5 这个请求的详细内容是什么
2. store2 的 resp.get_last_log_term()  为什么抽风了

我们回到 store2 上，
发现很有意思的事情—— raft 在处理请求的时候 resp 直接给设置了当前的 lastLogTerm_，
而我们看到 store2 当前的 lastLogTerm_ 刚好是 83:

```txt

(rr) l 1397
1392                                     << ", local lastLogId = " << lastLogId_
1393                                     << ", local lastLogTerm = " << lastLogTerm_
1394                                     << ", local committedLogId = " << committedLogId_
1395                                     << ", local current term = " << term_
1396                                     << ", wal lastLogId = " << wal_->lastLogId();
1397      std::lock_guard<std::mutex> g(raftLock_);
1398
1399      resp.set_current_term(term_);
1400      resp.set_leader_addr(leader_.host);
1401      resp.set_leader_port(leader_.port);
(rr) l
1402      resp.set_committed_log_id(committedLogId_);
1403      resp.set_last_log_id(lastLogId_ < committedLogId_ ? committedLogId_ : lastLogId_);
1404      resp.set_last_log_term(lastLogTerm_);
1405
1406      // Check status
1407      if (UNLIKELY(status_ == Status::STOPPED)) {
1408        VLOG(2) << idStr_ << "The part has been stopped, skip the request";
1409        resp.set_error_code(cpp2::ErrorCode::E_BAD_STATE);
1410        return;
1411      }
(rr) p lastLogTerm_
$80 = 83
```

作孽啊。再次观察 leader 发的 appendLog 请求：

```txt
(rr) p req.get_last_log_term_sent()
$32 = 81
(rr) p req.get_last_log_id_sent()
$33 = 12122
(rr) p committedLogId_
$34 = 12123
(rr)
```

leader 上的 log 情况呢：

```txt
(rr) up
#2  0x0000000004071423 in nebula::raftex::RaftPart::<lambda(std::shared_ptr<nebula::raftex::Host>)>::<lambda()>::operator()(void) const (
    __closure=0x7f5e8657e7d0) at /root/src/nebula/src/kvstore/raftex/RaftPart.cpp:785
785                                     eb, currTerm, lastLogId, committedId, prevLogTerm, prevLogId);
(rr) p prevLogTerm
$41 = 82
(rr) p prevLogId
$42 = 12124
(rr)
```

leader 上的 log index 其实已经到了 12124，
但是给 follower 发的却是 12122，
而 follower 上的 committedLogId_ 已经到了 12123。
这里的问题是什么？
Raft 算法能保证当选的 leader 包含所有已经 commited 的 log，
所以它在 12123 的时候必然能和 store2 上的日志匹配，
它为什么把 store2 的 last_log_id_sent() 滑到 12122 上了？
这是个大问题，我们的 raft 实现在 log match 上肯定是有问题的。
