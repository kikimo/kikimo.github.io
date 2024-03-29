---
title: "Raft 成员变更"
date: 2022-05-25T09:47:38+08:00
draft: true
---

关于 Raft 成员变更的几个问题：

1. 直接切换成员配置有什么问题?
2. 新增节点上的成员配置应该如何初始化，为什么？
3. raft 成员配置信息是否需要持久化，可否集中存储所有实例共享一份成员配置，为什么？
4. raft 快照信息是否需要包含成员信息？为什么？
5. 描述 join concensus 的完整流程
6. join concensus 如何保证 raft 数据安全，为什么需要 C(old, new) 这样一个中间配置状态
7. 单步变更如何保证 raft 数据安全
8. 单步变更在逻辑上有什么缺陷
9. raft 论文里为什么规定在 join concensus 执行中，看到一条配置变更的 log 需要立即 apply 让它生效
10. 如果我们把配置 apply 的时间调整改配置条目 commit 之后会有什么问题?
11. 在一条配置变更尚未结束的时候提交了另外一条配置变更会有什么问题，为什么?
12. 如果规定一条配置变更结束前不能提交另外一条配置变更，如何设计 raft 以实现该约束
13. 如果规定一条配置变更结束前不能提交另外一条配置变更，那么是否可以连续提交两条变更，为什么
14. 如果一个两节点集群，连续成功添加的三个新节点，有没有可能这三个新节点选举出一个新 leader 然后覆盖了之前两个节点上已提交的数据？为什么？
15. 如果一个节点收到一条 RPC 请求，该请求的来源节点不在它的 peer 列表里，可否直接丢弃该请求？为什么？
16. 为什么需要 learner 这个角色，没有 learner 配置变更操作中可能会出现什么问题？
17. 如果一个节点是 candidate，这时候收到一条成员变更的 log，对选举结果可能有什么影响？


## 1. 直接切换成员配置有什么问题

会脑裂。

## 2. 新增节点上的成员配置应该如何初始化，并阐述理由

新增节点上的配置有几种选择：空配置、C(old)、C(old, new)、C(new)。
首先绝对不能是 C(new) 的配置，会脑裂。

考虑空配置：新增节点可以等 leader 发快照过来后用快照里的配置信息。
不过，如果 leader 上的 log 还没有被截断就不需要发快照，这时候就没法获得正确的配置信息了。

## 6. join concensus 如何保证 raft 数据安全，为什么需要 C(old, new) 这样一个中间配置状态

Raft 算法的核心在于共识的达成。共识意味着超过半数节点达成一致，且达成的共识不会被推翻。
在 Raft 只要保证不脑裂，集群的数据就是安全的——既达成的共识不会被推翻。

join concensus 的配置状态变更:

1. C(old) -> C(old, new)
2. C(old, new) -> C(new)

集群中肯跟同时存在的配置变更：

1. C(old)
2. C(old), C(old, new)
3. C(old), C(old, new), C(new) (leader propose C(new) after C(old, new) was committed)
4. C(old, new), C(new)
5. C(new)

为什么需要 C(old, new) 中间状态？
C(old, new) 的引入保证集群不出现脑裂。
逐一分析以上几种情形：场景 1 和场景 5 集群的配置完全一致，
所以不会有脑裂问题。
对于场景 2，用反证法证明不会有脑裂问题：
假设集群在 Term(t) 脑裂了，假设某个 leader1 面的配置是 C(old)，
那么另一个 leader2 的配置必定是 C(old, new)(因为 leader1 已经拿到 C(old) 中节点在 Term(t) 的多数票)，
而 leader2 的配置也可能是 C(old, new)，因为它必须同时取得 C(old) 和 C(new) 中的节点的多数票。
因此，场景 2 不会有脑裂问题，同理可证场景 4 也不会出现脑裂问题。
那么接下来还要做的就是证明场景 3 也不会有脑裂问题。
场景三种集群的节点可以看到三种配置，C(old)、C(old, new)、C(new)。
根据 join consensus 的要求，
C(new) 出现的前提是 C(old, new) 已经复制到过半数节点，
此时如果一个节点上生效的配置还是 C(old) 的话，它将不可能当选 leader，
因为有过半数的节点日志都比它新。
所以我们只要论证集群在 C(old, new)，C(new) 这两种配置下不会产生脑裂即可，
而这和场景 2 的论证是一样的。


## 10. 如果我们把配置 apply 的时间调整改配置条目 commit 之后会有什么问题

配置变更 commit 前 apply 和 commit 后 apply 有一个比较大的区别是：
commit 前 apply 相比 commit 后 apply，C(old, new) 的提交需要同时也复制到 C(new) 囊括的集群中。
commit 后 apply 相比 commit 前 apply 有一个好处是不用担心 log rollback 带来的配置回滚问题。


## 17. 为什么需要 learner 这个角色，没有 learner 配置变更操作中可能会出现什么问题？

learner 主要用来同步数据的。leader 会向 learner 复制数据，但

1. leader 不把 learner 算在复制的 quorum 集里。
2. learner 没有投票权

配置变更的结束以 C(new) 的提交为标志。
这也意味着完成配置变更之前需要把 C(new) 和 C(new) 之前的所有数据都复制过半数的 C(new) 节点上。
这就有两个问题：

1. 当集群存在大量数据时，数据的复制会耗费大量时间
2. 同时在复制完成之前，集群将无法写入新数据

