---
title: "Raft Stale Read 问题分析"
date: 2022-05-17T21:00:15+08:00
draft: false
---

这几天在测试一份 Raft 代码的时候发现一个测试过不了：
客户端读不到刚刚写入成功的数据。
这套 Raft 为了保证先行一致性，
所有的读取操作也通过状态机达成一致，
所以出现这种 stale read 的情况就让人很意外。
加几行日志，把所有 Raft apply log 都打印出来，
这套 Raft 代码会批量 apply committed log，
在日志也打印了一次批量 log apply 的开始和结束。
在日志观察到一个情况略神奇：
出现 stale read 时，
总会在一次批量的 log apply 中发现一条 put 操作紧跟着一条 get 操作，
而这条 put 操作的 key/value 就是 stale read 中 miss 的那条记录。
review 代码，发现，在一次批量 Raft log apply 中，
用的是同一个 WriteBatch，
而这中间是允许读操作插进来的。
所以，导致 stale read 的一个合理解释就是 put 操作和紧连的 get 操作塞到同一个批量 raft log apply 操作中，
且他们操作统一个key/value，
put 写完但是 WriteBatch 还没提交，
自然后面的 get 也就读不到数据。

这个解释看起来很合理，但是客户端里只有确认一条请求已经成功提交后才会发起下一条请求，
如此一来，get 操作跟在 put 操作后面而且跑到一个 Raft batch log apply 流程里怎么解释呢？
我们在日志中发现，发生 stale read 这个时间点，测试集群出现网络分区了。
分区后选举出的新 leader 在一次批量 Raft log apply 中执行了出问题的 key/value 的 put 和 get 操作。
检索更早的日志，我们发现在出现网络分区，出问题的 key/value 已经在旧的 leader 上 apply 成功。
那么，客户端的 stable read 就可以这样解释：

1. old leader apply put(key1, value1)，回应客户端请求提交成功
2. old leader 被隔离，选举出 new leader，new leader 上的这条 key/value 数据尚未 apply
3. 客户端提交一条 get(key1) 请求给 new leader
4. new leader commit 这条 log，然后在一次 Raft batch log apply 种同时 apply put(key1, value1) 和 get(key1)
5. new leader 给客户端返回错误的 get(key1) 结果
