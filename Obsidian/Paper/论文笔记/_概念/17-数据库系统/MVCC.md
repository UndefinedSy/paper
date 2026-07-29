---
type: concept
aliases: [Multi-Version Concurrency Control, 多版本并发控制]
---

# MVCC

## 定义

Multi-Version Concurrency Control，通过为同一逻辑记录保留多个提交版本，使读事务获得一致快照，同时避免读写彼此阻塞。

## 核心要点

1. 读事务按时间戳选择可见版本，更新创建新版本。
2. 长事务会延迟旧版本回收，形成长版本链和空间/I/O 放大。
3. 图 OLAP 可持续数十分钟，而 OLTP 高频更新会快速制造大量版本。
4. [[ByteGraph-Dione]] 通过长 OLAP 聚合与短 OLTP 内存版本窗口降低版本保留成本。

## 代表工作

- [[ByteGraph-Dione]]：将长短事务的版本生命周期分开管理。
- [[Bigtable]]：以时间戳版本支持多版本数据访问。

## 相关概念

- [[Snapshot Isolation]]
- [[Lightweight Transactions for Hybrid Workloads]]
- [[HTAP]]
