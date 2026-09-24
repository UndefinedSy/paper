---
type: concept
aliases: [Continuous Query, Continuous Queries, 连续查询]
---

# Continuous Query

## 定义

一种持续运行的查询范式：查询不在单个时间点执行后关闭，而是在数据源持续变化的过程中不断报告最新的查询状态，属于流式数据处理的核心抽象之一。

## 核心要点

1. **与一次性查询的区别**：普通查询在某个时间点对数据库快照求值后返回，连续查询则是"永久订阅"，每当底层数据变化时自动重新求值或增量更新。
2. **常见约束**：学术系统（如 NiagaraCQ、CQL）通常要求查询满足某些限制（窗口函数、仅追加数据）以保证增量可计算性；Firestore 通过限制 filter 能力（至多一个关系运算符、无 join/聚合）来保证每条变更的增量结果都可直接从变更流计算，无需全量重扫。
3. **与流处理的关系**：流处理（如 Apache Storm、Millwheel）用任意代码替代查询，原理类似但表达能力更通用；连续查询保留 SQL-like 的声明式语义。
4. **与物化视图的区别**：物化视图在事务内与数据库保持一致，连续查询可以跳过某些更新后从当前时间点重跑，不需要事务一致性。
5. **Firestore 实现**：用 [[ChangeStream Filter]] 实现 $O(\text{matches}) + O(\log \text{queries})$ 的高效增量匹配，用 [[Safetime]] + [[Global Snapshot]] 提供跨查询的强一致时间戳语义。

## 代表工作

- Shivnath Babu and Jennifer Widom. Continuous Queries over Data Streams. ACM SIGMOD Record, 2001.
- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[Realtime Query]]
- [[Materialized Views]]
- [[Incremental View Maintenance]]
- [[ChangeStream Filter]]
- [[Global Snapshot]]
