---
type: concept
aliases: [wal2delta, WAL 转 Delta]
---

# wal2delta

## 定义

[[Lakebase]] 引入的 [[PostgreSQL]] **逻辑解码扩展**：挂接到逻辑复制流上，把 [[WAL]] 记录解码后直接以 **SCD Type 2** 格式追加到 Delta 表，实现从 OLTP 到 [[Lakehouse]] 的原生变更同步（forward-ETL）。

## 核心要点

1. **机制**：hook 进 PostgreSQL logical replication stream，逐条解码 WAL 记录并写成 Delta 追加。
2. **SCD Type 2 格式**：保留每个键的历史版本（而非只保留最新值），因此下游能消费完整变更历史。
3. **下游用法**：Spark 作业可消费历史流做复杂变换，或按需把它压实（compact）成快照。
4. **对比其他路径**：
   - 与 [[Direct-to-Storage]] 相反方向：DTS 解决 **Lakehouse → Lakebase** 的大批量装载；wal2delta 解决 **Lakebase → Lakehouse** 的持续变更流出。
   - 与 Direct Access 的区别：Direct Access 让分析引擎**直接读同一份页文件**（零拷贝但只读最新一致快照）；wal2delta 是把变更**物化**成湖格式的历史流。
5. **生态意义**：让 PostgreSQL 的变更流入湖仓而不需要外部 CDC 中间件，是"统一事务与分析"的两条互补通路之一。

## 代表工作

- [[Lakebase]]: 与 Direct Access 并列的分析集成的两条路径，分别面向数据管道与实时分析。

## 相关概念

- [[Change Data Capture]]
- [[Lakehouse]]
- [[Direct-to-Storage]]
- [[WAL]]
- [[Logical Revert]]
