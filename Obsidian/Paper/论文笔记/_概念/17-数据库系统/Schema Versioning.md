---
type: concept
aliases: [schema 版本化, 在线 Schema 变更]
---
# Schema Versioning
## 定义
让 schema 的多个版本在集群中共存并渐进演进的机制，使分布式数据库能在不停服、不全局加锁的前提下完成 DDL。
## 核心要点
1. 核心难题：分布式集群中各节点无法瞬时同步 schema，直接切换会导致节点间对数据格式的理解不一致。
2. [[F1]] 的方案是引入中间状态（delete-only / write-only），保证任意时刻至多两个相邻版本共存且互相兼容。
3. Azure SQL 的 MD-MVCC 进一步把 MVCC 应用到 schema 变更本身。
4. [[Aurora Limitless Database]] 把它列为「PostgreSQL 最需要现代化的地方」之首——高可用分布式应用还需要 canary 发布与 schema MVCC。
5. Limitless 现状：DDL 通过「所有 shard + 所有 router 加锁 + 扩展版 2PC」保证原子可见，代价是要阻塞并发 DML。
## 代表工作
- Online, asynchronous schema change in F1 (VLDB 2013)。
- MD-MVCC: Multi-version concurrency control for schema changes in Azure SQL database (VLDB 2025)。
## 相关概念
- [[F1]]
- [[MVCC]]
- [[Two-Phase Commit]]
- [[AuroraLimitless]]
