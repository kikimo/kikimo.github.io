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