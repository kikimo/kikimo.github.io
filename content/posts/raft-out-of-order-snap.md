---
title: "消息乱序对 Raft 快照的影响"
date: 2023-03-04T14:01:45+08:00
draft: false
---

Raft Leader 向 Follower 传输快照的时候，乱序的消息可以能会导致集群瘫痪，看以下例子：

![out-of-order-snap](/images/out-of-order-snapshot.jpeg)

1. Leader 向 Follower 先后发送了 sendAppend(130, 132)(发送 index 在 [130, 132] 范围内的日志)和 sendAppend(133, 135) 两条 RPC
2. 这两条消息在网络传输中乱序了，导致 Follower 先收到 sendAppend(133, 135)，然后才收到 sendAppend(133, 135);
3. Follower 收到 sendAppend(133, 135) 后 Reject 这条消息，然后又 Accept 了 sendAppend(130, 132) 的消息
4. Leader 在收到 Follower 的 Reject 回应前把日志压缩到 131 位置，收到 Follower Reject 消息触发 Leader 向 Follower 发送快照 Snap(131)(快照中的 lastAppliedIndex = 131)，并把 Follower 状态设置为等待快照接受状态
5. 然后 Leader 收到 sendAppend(130, 132) 消息的 Accept 回应，把 Follower 的 matchIndex 跟新到 132
6. Follower 收到 snap(131) 更新了本地快照，并把重置日志信息——把最后一条日志设置为 131
7. 之后 Follower 的给 Leader 回应的 commitIndex 都是 131 < 132(matchIndex)，Follower 无法从等待快照状态中恢复过来，再也收不到新的日志信息

解决方法：

1. 当 Follower 处于等待快照状态时 Leader 忽略所有 sendAppend 响应消息，这样就可以防止在以上步骤 5 中 Leader 更新了 Follower 的 matchIndex
2. 第二种方法稍微复杂一点：
    + 当 Leader 收到 sendAppend 并且可以更新 matchIndex 时，如果监测到 Follower 处于等待快照状态中，则把 Follower 重置为正常的日志收发状态
    + 当 Follower 收到快照消息 snap(index) 的时候，如果快照有效，那么在更新快照的同时，保留本地 index + 1 的所有日志
