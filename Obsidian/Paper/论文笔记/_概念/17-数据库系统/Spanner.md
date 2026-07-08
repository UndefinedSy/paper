---
type: concept
aliases: [Google Spanner, Spanner Database]
---

# Spanner

## 定义

Google 的全球分布式数据库系统，提供跨数据中心的 ACID 事务和强一致性保证，是 NewSQL 运动的代表系统。

## 核心要点

1. 提供全球分布式 ACID 事务，使用 TrueTime API
2. 强一致性（外部一致性），与 [[Bigtable]] 的 [[Eventual Consistency|最终一致性]] 形成对比
3. NewSQL 运动代表，融合 SQL 和 NoSQL 的优势
4. 覆盖 Bigtable 不支持的跨行 ACID 场景
5. 与 CockroachDB Serverless 等系统类似

## 代表工作

- Spanner: Google's globally-distributed database (OSDI 2012)

## 相关概念

- [[Bigtable]]
- [[Eventual Consistency]]
- [[HBase]]
- [[Cassandra]]
