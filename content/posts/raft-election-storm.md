---
title: "Raft 中的选举风暴"
date: 2021-10-30T19:23:36+08:00
draft: false
---

>A server remains in follower state as long as it receives valid RPCs from a leader or candidate. 

Follower 在 election timeout 期间内如果收到合法的心跳包或者选举请求就会继续保持 follower 状态。
那么，什么是合法的选举请求？一个合法的选举请求必须满足一下两个条件：

1. 选举请求中的 term 满足 term >= currentTerm
2. 当前节点中 votedFor 字段为 null 或者等于 candidateId

当 term > currentTerm 时需要把节点转为 follower，同时设置 votedFor 字段为 null。
什么情况下 term == currentTerm && votedFor == null？
当 Raft 第一次运行的时候，此时各个实例的 term == 0，且 votedFor == null。
什么情况下 term == currentTerm && votedFor == candidteId？
当 raft 实例收到 candidate 重复的 RPC 请求，
此时可能 term == currentTerm 且 votedFor == candidateId，
此时 follower 也要投票给 candidate。

为什么收到合法的 candidateId 时 raft 实例也要保持 follower 状态——即重置 election timeout 计时器？
如果不这样的话可能出现刚刚选举完 leader，follower election timeout 又发起新一轮选举把它踢掉。
这会让 raft 集群不稳定，
同时也可能产生选举风暴 —— raft 集群中刚当选的 leader 不停的被超时的 follower 踢掉而陷入无休止的选举循环中。

>If many followers become candidates at the same time, votes could be split so that no candidate obtains a majority. When this happens, each candidate will time out and start a new election by incrementing its term and initiating another round of RequestVote RPCs. However, without extra measures split votes could repeat indefinitely.

当选举由于平局时，candidate 会发起下一轮选举。
当平局出现的时候——也就是每个 candiate 都收到所有 peer 的选举答复，
他们发现自己没有得到足够的选票，
并且自己还是 candidate 状态（没有收到合法 leader 的心跳包）
这时候 candidate 就要进入下一轮选举。
然而下一轮的选举不能马上开始，
如果 candidate 在知道平局结果时就马上开始下一轮选举就也可能陷入选举风暴：
candidate 太多而导致选举平局，马上进入下一轮选举，又是选举平局，循环往复怎么都选不出 leader。
正确的做法是等到 election timeout 后才开始下一轮选举。
选举中的 election timeout 和 follower 的 election timeout 一样时选举一段时间范围内的随机值。

