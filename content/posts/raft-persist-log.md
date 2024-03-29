---
title: "Raft 日志持久化"
date: 2022-01-09T13:53:52+08:00
draft: false
---

Raft 日志需要持久化，问题是在什么时候持久化？
用户每提交一条日志都要先保证落盘吗？
这种做法可以保证数据的完整性，
但想象一下每次都是单条日志进来，每条日志都触发`write()`、`fsync()`等操作，
如此一来性能肯定不理想。
我们可以等一批日志进来后再批量落盘。
但还是要回答一个问题，什么时候需要保证日志数据落盘？
或者我们不妨换个角度考虑这个问题：如果日志数据没有及时落盘会出现什么问题？
我们知道，一条日志包括几种状态：

1. uncommited：未提交
2. commited：已提交
3. applied：已执行

先考虑已提交的日志——如果一条日志已经提交但是没有落盘会出现什么问题？
我们假设在一个五节点的 Raft 集群中出现的一个场景：

1. p1 是 leader，p1 成功将一条日志 l1 复制到 p2、p3
2. p2、p3 感知到 l1 被提交，并在本地执行这条日志
3. p1 尚未执行 l1，且 l1 在 p1 上尚未落盘，这时候 p1 crash
4. p1 重启，p4、p5 两个节点选举 p1 为 leader
5. p1 在原来 l1 日志对应的 index 提交一条新日志 l2

可以看到日志 l2 和原来的 l1 冲突了，而不幸的是 l1 在 p2、p3 上已经被执行了。
通过以上分析可以知道一条日志在提交之前一定要保证落盘，
因为日志的执行在日志提交之后，
所以如果我们保证这一点那也就意味着一条被执行的日志肯定已经落盘。

不考虑快照、日志压缩的情况下，Raft 要保证数据不丢失只要保证提交的日志数据不丢失即可。
因为对于尚未明确需不需要提交的日志数据，及时丢了也不会影响到算法的正确运行。
综上，在实现 Raft 的时，
我们可以在确定一批日志数据可以提交的时候再执行批量落盘操作，
这样既保证算法的正确运行又提升了算法的效率。
