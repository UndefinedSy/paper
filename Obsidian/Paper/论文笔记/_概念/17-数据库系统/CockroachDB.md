---
type: concept
aliases: [CRDB]
---
# CockroachDB
## 定义
开源的弹性地理分布式 SQL 数据库，以 Raft 复制 range、以 HLC 定序。
## 核心要点
1. 支持 serializability 与 [[Read Committed]]，但**不支持 [[Snapshot Isolation]]**。
2. 用 [[Hybrid Logical Clock]] 为分布式事务确定 commit 时间戳。
3. **一般不保证 [[External Consistency]]**（官方 consistency model 博客有明确说明）——这是与 [[Spanner]] 及 [[Aurora Limitless Database]] 的关键差异。
4. 面向跨 region 部署，依赖 Raft 共识；而 Limitless 面向跨 AZ，直接复用存储层从而无需额外共识。
## 代表工作
- CockroachDB: The resilient geo-distributed SQL database (SIGMOD 2020)。
## 相关概念
- [[Hybrid Logical Clock]]
- [[Serializability]]
- [[External Consistency]]
- [[YugabyteDB]]
