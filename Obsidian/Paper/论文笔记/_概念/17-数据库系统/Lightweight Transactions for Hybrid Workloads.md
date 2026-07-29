---
type: concept
aliases: [混合负载轻量事务, Long-Transaction Coalescing]
---

# Lightweight Transactions for Hybrid Workloads

## 定义

ByteGraph-Dione 针对图 [[HTAP]] 的版本管理机制：按窗口聚合长 OLAP 事务，并让短 OLTP 历史版本仅在内存保留有限时间。

## 核心要点

1. 多跳长查询批量执行，窗口内只持久化必要快照版本。
2. OLAP persistent version 可在内存紧张时换出到共享存储。
3. 短 OLTP 立即执行，独立版本默认只在内存保留 30 秒。
4. 30 秒聚合减少 59.44% 历史版本，结果差低于 0.00035%。
5. 相对 RC 的 SI 吞吐损失从 35.5% 降至 16.49%。

## 代表工作

- [[ByteGraph-Dione]]：提出并验证该长短事务分治机制。

## 相关概念

- [[MVCC]]
- [[Snapshot Isolation]]
- [[HTAP]]
