---
title: "Raft 笔记（二）—— 日志复制"
date: 2021-01-16T21:38:04+08:00
draft: false
---

## 1. 日志复制算法

[Raft 笔记（一）—— 选主](https://coderatwork.cn/posts/notes-on-raft-1/) 中我们提到了 AppendEntries RPC，
当时我们把它当成发送心跳的 RPC，
这个 RPC 的另外一个核心用途是日志复制，
这篇文章主要讲 Raft 三个重要模块中的日志复制模块。

Raft 中的每个节点都维护一个日志列表，
客户端向主节点追加日志，
日志复制暨主节点把日志复制到其他节点上，Raft 会记录每条日志对应的任期信息。
当主节点把一条日志复制到超过半数的节点上后，我们变可以认为这条日志被成功提交了。

Raft 保证日志在复制过程中始终遵循几个性质：

1. 主节点只追加日志：主节点不会覆盖或者删除已有的日志
2. 日志匹配：两个节点上同一个索引位置中的日志，如果他们的 term 一样，那么这两条日志便是一样的，且自这个位置之前的所有日志也都是一样的
3. 主节点完备性：一条被提交的日志也会出现在后续所有的主节点的日志列表中（且位置不变）

主节点的完备性保障了 Raft 分布式共识算法的正确性。
Raft 的主节点除了必须获取超过半数的投票外还有一条非常重要的约束——只有当 candidate 上的 log 至少不比自己旧的时候，
follower 才会把票投给它。
Raft 中对这里更新的日志的定义是：1. 最后一条日志的 Term 更大 2. 或者 Term 一样，但是 len(log) 更大。
Raft 论文中提到这条性质可以保证选举出来的主节点总是包含所有被提交的日志。
但是论文中没有论证这个结论。
这里我们提供一下大致的证明思路。
首先我们定义 order() 函数来描述节点上日志的更新成都，
例如 order(S1.log) > order(S2.log) 表示 S1 上的日志比 S2 新。
我们通过反证法来论证，
首先我们假设 order(S1.log) > order(S2.log)，
S2 中存在已提交的日志且改日志没有出现在 S1 中，
我们记 S2.log[i] 为第一条这样的日志，
我们只要能证明 S2.log[i].Term > S1.log[i:].Term 便可得出和 order(S1.log) > order(S2.log) 矛盾的结果。
首先因为 S2.log[i] 是一条已提交的日志，
所以 S2.log[i] 已经成功复制到过半数的节点上，
那么我们显然有 S1.log[i].Term < S2.log[i].Term（
执行 AppendEntries RPC 到 S1.log[i:] 的 Leader 的 Term 必然小于 S2.log[i].Term，
否则它无法当选）；
如果 len(S1.log) = i，那么我们就完成证明了；
假设 len(S1.log) = i + 1，
因为 S1.log[i].Term < S2.log[i].Term，
那么显然 S1.log[i+1].Term < S2.log[i].Term 也成立，
否则 S1 在 index i 位置上的 S1.log[i].Term 必须大于半数节点，
但这一点我们上面已经证伪。
对于 len(S1.log) > i + 1 的情况，
还是采用上面类似的证明套路，
再加以归纳法即可证明。

除了这条限制，Raft 还有另外一条非常重要的限制——主节点不提交（commit）更早起的 term 的日志。
Raft 论文通过一个反例来说明为什么主节点为什么要遵循这条规则：

![Raft 日志覆盖](/images/raft/log_commit_term.png)

在 Raft 论文举的这个例子中，
a 时刻 S1 是 term 2 的 leader，
他把 log[2] 复制到两个节点；
b 时刻 S1 挂了，
S5当选新主节点，
它在日志追加了 log[2]；
c 时刻 S5 挂了，
S1 重新当选，、
S1 把它的 log[2] 进一步复制到 S3 上，
此时 S1 上的 log[2] 已出现在超过半数的节点上，
假如此时 S1 把 log[2] 提交了，
然后它崩溃了，
然后 S5 又当选新的主节点
（注意到 S5 上 log[2] 日志项的 term 是 3，
比 S1、S2、S3 都要来的高，所以它可以当选主节点），
一旦 S5 当选主节点它上面的 log[2] 可以复制并覆盖 S1、S2、S3 上已经提交的 log[2]。
另一种情况假设 S1 在崩溃前追加并提交了 log[3](term 4)，‘
那么 S5 则连当选主节点的机会都没有，
也就不会发生上面描述的已提交的日志项被覆盖的情况。
Raft 论文中举的这个反例说明了提交早起 term 的日志项会导致已提交的日志项可能被覆盖的问题。
和前一条性质一样，
这个例子中出现的主要问题是 Raft 提交了就得 term 中的日志，
然后包含次新 term 中的日志的节点当选 Leader 然后把前面提交的旧 term 中的日志覆盖了。
Raft 算法限制只能提交当前 Term 内的日志保证了次新 Term 的日志不会把已提交的日志覆盖掉
（因为次新 Term 肯定比 currentTerm 小）。


最后，Raft 论文使用反证法来论证主节点的完备性：
首先假设 T term 内 Leader(T) 提交的某条日志未出现在 U term 的 Leader 日志中，其中 U > T，那么：

1. Leader(T) 必然已经把这条日志复制到超过半数的机器上
2. 必然存在某个选举 Leader(U) 的 Voter(X) 上存在这条日志
3. Voter(X)  投票给 Leader(U) 说明 Leader(U) 上的日志至少不比 Voter(X) 上的日志旧
4. 假设第一种情况 Leader(U) 和 Voter(X) 上最后一条 log 的 term 一样，
那么 Leader(U) 至少不比 Voter(X) 的 log 序列长度小，
Leader(U) 上必然包含 Leader(T) 已提交的这条日志（根据日志匹配原则），
这是第一种矛盾
5. 第二种情况 Leader(U) 上最后一条日志比 Voter(X) term 值大，
那么根据我们前面介绍的 Raft 主节点选举限制，
Leader(U) 也必然包含 Leader(T) 提交的日志，这是第二种矛盾情况

综上可知 Raft 算法种的 Leader 完备性成立。

## 2. 日志复制算法实现上的几点问题

1. 为什么 currentTerm、voteFor、logs 这几个字段需要持久化
2. Raft 持久化需要保存节点的角色吗（Leader、Follower、Candidate）？为什么？
3. 为什么 matchIndex、nextIndex 不需要持久化？
4. 论文中提到 matchIndex 的匹配可以优化但是没说具体如何优化，
实现中可以采用二分法寻找上确界的方法来加速寻找，
Follower 和 Leader 两边都可以根据输入参数用二分法来加速寻找匹配的日志项，
mit 的讲义中提到另外一种方法，但是没看懂 TODO
4. UT 真的很重要！这不算一个问题，这是一个教训！Unit test 真的非常非常非常重要，因为单元测试阶段 debug 的成本远远小于集成测试阶段。