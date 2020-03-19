---
title: "架构整洁之道"
date: 2020-02-23T14:07:21+08:00
draft: true
---

良好的架构具有以下特点：

1. Independent of frameworks. 架构不依赖于特定的框架。让框架回归工具属性，而不是让你的系统受制于框架。
2. Testable. 测试可不依赖于任何外部组件，诸如：数据库、UI、web 服务器。
3. Independent of the UI. 在不改变系统其它部分的情况下，灵活调整系统的 UI。
4. Independent of the database. 业务逻辑不应当依赖于某款具体的数据库。
5. Independent of any external agency. 系统的业务规则不应当依赖认为外部具体的技术细节。 

良好的架构本质上需要做好两个事情：

1. 架构分层
2. 处理好层次间的依赖

架构的层次由内到外、由核心到次要依次可分为：企业业务规则、应用业务规则、接口适配层、框架和驱动层。
层次间应当保证内层不依赖外层，只能让外层来依赖内层。

> Source code dependencies must point only inward, toward higher-level policies.

## 1. Entities

## 2. Use Cases

## 3. Interface Adapters

## 4. Frameworks and Drivers

## 5. Crossing Boundaries


