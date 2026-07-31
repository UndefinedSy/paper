---
type: concept
aliases: [Citus Data, Azure Cosmos DB for PostgreSQL]
---
# Citus
## 定义
基于 PostgreSQL 扩展实现的分布式数据库（现属 Microsoft Azure），支持把表分片到多个 worker 节点。
## 核心要点
1. 以 PostgreSQL extension 形式实现，兼容性好、侵入性小。
2. 通过 [[Two-Phase Commit]] 为分布式事务保证原子性、一致性与持久性。
3. **关键局限：不为分布式事务提供 [[Snapshot Isolation]]**，因而把用户暴露在大量事务异常之下——这是 [[Aurora Limitless Database]] 与它的核心差异。
4. 也不像 Limitless 那样同时提供垂直与水平两个维度的自动伸缩。
## 代表工作
- Citus: Distributed PostgreSQL for data-intensive applications (SIGMOD 2021)。
## 相关概念
- [[Sharding]]
- [[Two-Phase Commit]]
- [[Snapshot Isolation]]
- [[AuroraLimitless]]
