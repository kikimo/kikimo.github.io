---
title: "node_exporter 性能优化"
date: 2020-11-01T09:48:38+08:00
draft: true
---

k8s 集群中通常会部署 node_exporter 用于采集节点的基础信息。
在 k8s 集群运行一段时间后我们发现 node_exporter 占用的 CPU 明显增多。
