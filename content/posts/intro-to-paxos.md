---
title: "Paxos 极简介绍"
date: 2022-06-05T17:48:04+08:00
draft: false
---

Paxos 算法描述多个节点参与，对某个值达成一致的算法：

1. 一旦达成共识，结果不可推翻
2. 只要有过半数节点参与即可达成共识
3. 参与的节点可以习得已经达成的共识

Paxos 算法中定义了三种角色：proposer、acceptor、learner。
由 proposer 发起提议（proposal），
acceptor 决定是否接受 proposer，
learner 可以习得达成的共识。

如何决定达成共识？
如果只有一个 acceptor，
那么达成共识是一件很容易的事情——**我们总是 acceptor 接受收到的第一个 proposal 即可**。
但这就存在单点故障的风险。
所以系统需要设置多个 acceptor 来防止单点故障。
对于多个 acceptor 的场景，
达成共识的最简单方式就是通过多数派投票来达成共识。
只要有过半数的节点同意，我们就可以达成共识。
但这同样存在问题：
一个集群包含三个 acceptor，
且这三个 acceptor 分别接收了三个不同的 proposal，
这时候就形成一个死局了。
所以必须允许 proposer 多次提交 proposal。
paxos 的给每个 proposer 一个**全局唯一且单调递增的计数器**，
每次提交 proposal 都关联一个的计数编号 n，
同一个 proposer 保证给每个 proposal 赋以一个本地唯一的编号，
但可以和其他 proposer 设置的编号重复。
这样就可以让 proposer 多次提交 proposal。
但是 paxos 不能允许 proposer 随心所欲的提交 proposal。
如果当前以已经达成共识，
proposer 却通过递增 n 提交了一个新的 proposal，就可能导致已达成的共识被推翻。
如何解决这个问题？
**在共识已经达成的情况下**，
倘若每个 proposer 提交的 proposal 都是已达成的共识就能保证已达成的共识不会被推翻。
如何让每个 proposer 提交的 proposal 都是已达成的共识？
我们结合归纳法来找出保证共识不被推翻的条件。
假设 paxos 已达成共识 (proposal, m)，
m 为 proposal 关联的编号。
我们尝试归纳证明 n > m 提交的了一样的 proposal。
假设 [m, n - 1] 编号范围内提交的是一样的 proposal。
一个 proposal 被接受的前提是有过半数的 acceptor 接受它，
既然该 proposal 已被接受，那么必然存在超过半数的节点上存在这个 proposal 且这个 proposal 的编号范围在 [m, n-1] 之间。
这推论之所以成立的原因是：

1. proposal 首次被接受时的编号是 m，此时集群中即存在过半数的节点包含 (proposal, m)
2. [m, n - 1] 范围内提交的都是同一个 proposal，他们有可能覆盖原来的 (proposal, m)，只是改变了 proposal 的编号

如此以来，要保证提交 (proposal, n) 而不破坏共识，我们只需要找出已提交的这条 propsal 即可。
如何找出这条 proposal？
首先我们先**假设 n 比当前集群中已接受的 proposal 的 编号大**，
那么**向过半数的 acceptor 节点发请求咨询当前已接受的 proposal，编号最高的那条 proposal 即是我们要找的 proposal**。
这里需要回答两个问题：

1. 为什么是向过半数的节点发请求咨询
2. 为什么编号值最高的 proposal 就是已提交的 proposal

第一个问题其实很好回答，因为我们假定当前已经达成共识，
所以系统必然存在一个集合包含过半数的节点，他们接受了 proposal，且编号范围在 [m, n-1]，
只要咨询过半数的节点就一定能找到一个节点在这个集合里。
第二个问题嘛，其实也好回答。
反证法，因为我们已经回答了在一个过半数 acceptor 节点的集合里必然有一个节点已经接受 proposal，
假设这个 proposal 的编号 s 不是集合中最大的那个，根据我们归纳法中假设，
proposer 提交的大于 s 的都是同一个 proposal，显然编号最大的也应该是已提交的那个 proposal。

以上证明成立的一个基础是 [m, n-1] 范围内提交的都是同样的 proposal。
如果 proposer 在提交了 m 编号的 proposal 后在提交一个 m -1 编号的 proposal 呢？
这显然会证明成立的基础，因此还需要额外加一条限制：**acceptor 只接受编号严格单调递增的 propsal**.

现在可以考虑如何实现 paxos 算法了，论文里把 paxos 算法的实现分成两个阶段，prepare 阶段和 accept 阶段：

### 1. prepare 阶段

1. proposer 选择编号 n 向 acceptor 提交 prepare 请求
2. acceptor 处理 prepare 请求
    + 如果 prepare 请求的编号 n <= acceptor 已知的 proposal 编号，则拒绝该 prepare 请求
    + acceptor 返回它已知的 proposal（如果没有则返回空） 

### 2. accept 阶段

1. 如果 propser 收到过半数的 preppare 请求响应且没有被拒绝，则：
    + 将本地的 proposal 这是为 prepare response 中编号最高的 proposal，如果 prepare response 返回的 proposal 都不为空的话
    + 向 acceptor 发送编号 n 的 accept 请求(proposal, n)
    + 如果有过半数的 acceptor 接受 accept 请求，则认为达成共识
2. acceptor 处理 acceptor 请求
    + 如果 prepare 请求的编号 n <= acceptor 已知的 proposal 编号，则拒绝该 prepare 请求，否则，接受该请求
