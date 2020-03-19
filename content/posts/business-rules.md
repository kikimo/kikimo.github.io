---
title: "关于业务规则"
date: 2020-02-23T15:03:24+08:00
draft: true
---

**Clean Architecture Chapter 20 Business Rules 笔记**

业务规则就是能一组能创造价值（帮公司赚钱、省钱）的流程。
业务规则可能由人或计算机软件来协助完成。
业务规则经常依赖某些数据，这些数据称为业务数据。
业务规则和业务数据是密不可分的，它们经常被封装到对象中，也是就是我们常说的业务实体（Business Entity）。

## 业务实体

实体是对特定业务规则及其业务数据的封装。
实体内部通常存有业务数据，或者可以通过实体便捷的访问到业务数据。
实体方法用于实现相关业务规则。

## 用例

不是所有的业务规则都能封装在业务实体中。

Some business rules make or save money for the business by defining and constraining the way that an automated system operates.
These rules would not be used in a manual environment, because they make sense only as part of an automated system.

放贷应用：
1. 收集验证用户姓名、联系信息
2. 收集用户评分
3. 根据评分结果决定用户贷款
