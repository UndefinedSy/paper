---
title: "ACID"
category: "数据库系统"
tags: [transactions, consistency]
created: 2026-08-07
---
# ACID

[[ACID]] 指数据库事务的原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability）保证。

## 在 GaussDB-Vector 中

[[GaussDBVector]] 以事务引擎保证新插入向量在提交后立即可见，并在逻辑删除、图 vacuum 与页回收时协调并发读写，令实时 ANN 维护不破坏关系数据库事务语义。

## 关联
- [[MVCC]]
- [[GaussDBVector]]
