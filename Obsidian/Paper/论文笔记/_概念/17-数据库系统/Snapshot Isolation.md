---
type: concept
aliases: [SI, 快照隔离]
---

# Snapshot Isolation

## 定义

一种基于 [[MVCC]] 的事务隔离级别：事务读取开始时刻的一致快照，通常只在提交阶段检测并发写冲突。

## 核心要点

1. 避免读写互相阻塞，适合长时间分析查询。
2. 需要保留事务快照仍可能访问的历史版本。
3. 长 OLAP 与高频 OLTP 更新并存时，版本保留成本会迅速增长。
4. SI 通常避免脏读和不可重复读，但不等同于完全可串行化。

## 代表工作

- [[ByteGraph-Dione]]：以事务聚合降低图 HTAP 中 SI 的版本和吞吐开销。

## 相关概念

- [[MVCC]]
- [[Lightweight Transactions for Hybrid Workloads]]
- [[HTAP]]
