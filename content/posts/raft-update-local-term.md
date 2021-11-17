---
title: "为什么 Raft 遇到 higher term 要 update 本地 term"
date: 2021-11-17T21:49:32+08:00
draft: false
---

我们假设一个三节点 raft 集群中有 r1、r2、r3 三个实例，出现以下情况：

1. leader r3 宕机了
2. r2 首选发起 election，但是 r2 上的 log 没有比 r1 更新，所以 r1 不会投票给 r2，r1 也不更新本地的 term 直接返回了
3. r1 超时发起选举，但是 r1 的 term 一直比 r2 小（r2 先发起的投票，term 增长比 r1 快，同时 r1 收到 r2 的请求也不会 reset 本地 term），所以 r1 也没法当选 leader

r1、r2 可能陷入选举死循环永远都没法选举出 leader。
