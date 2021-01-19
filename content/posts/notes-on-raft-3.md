---
title: "Raft 笔记（三）——快照"
date: 2021-01-19T21:51:01+08:00
draft: false
---

Raft 的 log 保存在内存中，
内存无法支撑 log 无限增长，
因此 Raft 用快照技术将某个时间段前的状态转成快照并存储在次盘上，
这个时间段之前的的 log 就可以删掉了。
Raft 快照存储的是**从 log 开始到具体某个时间点下的已经提交的日志的状态机执行结果，而不是具体的 log 内容**。
除此以外 Raft 快照中还包含两个字断 lastIncludedIndex 和 lastIncludedTerm，
这两个字段在 AppendEntries RPC 调用时用来执行一致性检查，
例如当我们网快照之后的 log append 日志时可能就需要检查快照中最后一条日志的 inddex 和 term 是否匹配。
Raft 中的每个节点都会执行快照转存操作，
因为快照转存针对的都是已经提交的日志，
所以我们不用担心数据不一致的问题（Raft 算法已经保证了节点间已提交日志的一致性）。
节点在什么时候执行快照转存操作？
Raft 论文中没有限定具体的执行时机，
但给出一个简单的策略：当内存中 log 日志的大小达到某个固定值的时候便执行快照转存。

Raft 为快照处理添加了一个新的 InstallSnapshot RPC.
通常节点自己做快照转存就可以了，
但在某些特殊情况下，
节点的日志可能大幅落后于主节点，
比如这是一个新增的节点，
或者由于网络延迟导致节点数据落后较多，
这时候主节点就会向这些节点调用 InstallSnapshot RPC。
Leader 如何判断 Follower 的日志落后太多，
也就是它如何具体判断 InstallSnapshot RPC 的调用时机？
当 Leader 已经将部分 Log 丢弃（也就是这部分内容已经转存成快照了）时，
如果它发现 Follower 上还没同步这部分数据，
它就会发起 InstallSnapshot RPC。
InstallSnapshot 可能会把快照分多次发送，
所以 RPC 接口参数里有个`done`字段表示快照是否已经传输完毕，
除此以外还有一个`offset`字段表示当前数据的偏移量——InstallSnapshot传输的都是二进制数据。
在实现 InstallSnapshot 的时候一个需要特别留意的地方就是，
RPC 可能是乱序抵达的，代码上的实现需要考虑这个问题。
Follower 在收到全部的快照数据后用新的快照替代当前的快照，
并且丢弃日志中已经包含在快照中的数据。

