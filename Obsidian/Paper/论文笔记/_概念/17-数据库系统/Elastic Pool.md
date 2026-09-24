---
type: concept
aliases: [弹性池]
---

# Elastic Pool

## 定义

Azure SQL 中多个数据库共享一份 CPU/IO 资源的部署形态，多个租户数据库在同一实例内竞争计算与存储资源。

## 核心要点

1. 在 Elastic Pool 中，单线程版本清理器容易成为瓶颈：多个数据库同时产生版本，单线程难以跟上聚合速率。
2. Elastic Pool 占 SQL Azure 相当大比例，因此并行化清理被认为是关键需求。
3. 动态并行清理仅在资源充足、PVS 超过阈值时才在库内/跨库启用，避免小体量数据库因清理而消耗不必要资源。

## 代表工作

- [[AzureMVCCVersionCleaner]]：以动态并行清理应对 Elastic Pool 场景下的版本清理瓶颈。

## 相关概念

- [[Persistent Version Store]]
- [[MVCC]]
