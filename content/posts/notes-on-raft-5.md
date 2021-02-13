---
title: "Raft 笔记（四）——客户端"
date: 2021-01-23T12:35:59+08:00
draft: true
---

Raft 算法的核心是通过共识算法保障多个节点上日志序列的一致性，
然后把这些日志作为 DFA 的输入，从而在多台机器上得到相同的结果。
在实际应用中，我们需要考虑 Raft 如何具体查询或者变更数据的问题，
这就是这篇文章要讲的内容。

Raft 通过客户端来查实现和服务端的交互。
客户端在初始化的时候会得到一个服务端的地址列表。
当需要和服务端通信的时候，Raft 可以查询任意一个服务端。
但是只有 Leader 可以对服务的请求作出回应，
如果客户端的查询对象不是 Leader，
它会在返回结果中告诉客户端那个节点才是 Leader。

服务端什么时候告知客户端操作失败？
Index 位置的数据是其他元素？

1. 如何处理重复请求
    + clientID + reqSeqID + clientMap

2. 如何保证读取到最新结果
    + leader 更新最新的 commit：当选时，提交一个 noop 的 log entry
    + 读取数据前，Leader 先发送 AppendEntries，确保数据是最新的

3. 服务端如何确定 append log 失败？
    + index，term log 不一致或者 lose quorum

4. 如果服务端还在执行呢？
