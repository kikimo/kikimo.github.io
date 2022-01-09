---
title: "Raft 选主问题——及时更新 currentTerm"
date: 2022-01-09T15:28:52+08:00
draft: false
---

Raft 中的 term 相当于 逻辑时钟，Raft 论文里要求：
当一个 Raft 实例在 RPC Request/Response 中看到比自己 currentTerm 更大的 term 时需要立即更新 currentTerm 并转为 follower。
不及时更新 currentTerm 的会有诸多问题，
我们考虑一种因为 term 更新不及时导致的选举死循环问题，
考虑一种一个五节点的 Raft 集群：

1. p3 是 term 408 的 leader
2. p4 发起 term 409 的选举，p2、p5 投票给 p4
3. p4 当选 term 409 的 leader 但是马上被网络隔离

此时 p3 还以为当前 term 是 408，
继续给 p2、p5 发日志。
p2、p5 拒绝它的日志，
但是发给 p3 的 RPC Response 没带自己的 term(此时值为 409) 或者
p3 没有根据 RPC Response 中的 term 更新自己的 currentTerm 和 role。
如果系统实现了 prevote，而且 p3 的日志比 p2、p5 新，
那么 p2、p5 的 prevote 会一直失败，p2、p5 的 prevote 请求失败了所以他们的 term 不会进一步增长，也无法发起正式选举；
另一方面 p3 不会更新自己的 term 且一直以为自己还是 term 408 的 leader，
但是 p2、p5 不认它的日志，这时候……emm……整个集群就悲剧了。

另一个涉及到 currentTerm 字段跟新和 Raft 选主的问题是：
在一个实现了 prevote 的 Raft 集群中，节点要不要根据 prevote RPC 中的 term 信息来跟新 currentTerm 字段？
或者如果不更新会有什么问题？
考虑一下场景，还是在一个五节点的集群中：

1. p1、p2 两节点被网络隔离
2. p3、p4、p5 三个节点之间发起选举，三节点的 currentTerm 字段分别为 116、116、115 且 p5 上的日志比 p3、p4 新
3. p5 不会再 prevote 中投票给 p3、p4，因为它的日志更新
4. p3、p4 也不会投票给 p5 因为 p5 的 term 比他们小

如果 Raft 节点不根据 prevote 字段中的 term 跟新自己的 currentTerm 的话，
在上述的场景中三个节点将永远也无法选举出 leader。

以上我们讨论的两个问题都涉及到 Raft 的 prevote 机制。
关于 prevote 还有一个比较重要的问题。
我们知道在正式的 vote 请求中，
一个 Raft 实例需要先给自己的 currentTerm 加一。
那么 prevote 前也需要递增 currentTerm 吗？
我之前一直认为可以这么做：只要在 prevote 中递增 currentTerm就可以了，
在正式的 vote 中就不需要再递增 currentTerm了。
这种做法一个明显的弊端是如果系统一直 prevote 失败，
那 currentTerm 也会一直递增，
而 prevote 目的之一就是要用来防止 currentTerm 的无序递增。
最近读到的一篇文章中[](https://dl.acm.org/doi/pdf/10.1145/3447851.3458739)，
这篇文章让我想到以上做法会产生的一个严重问题——选举死循环，考虑一个三节点的集群：

1. p1 和 p2、p3 单向隔离，既 p1 可以发消息给 p2、p3 但是 p2、p3 无法给 p1 发消息
2. p2 是 leader，因为 p1 总是收不到 leader 心跳所以发起 prevote

因为 p2 总是收不到 leader 心跳他会不停的发起 prevote，
如果我们在 prevote 时就递增 currentTerm 那么 Raft 集群中的 term 不停的递增，
p2、p3 因为 term 递增而陷入无限制的 leader 选举中。
综上，我们可以认为在 prevote 中不应该递增 currentTerm。



