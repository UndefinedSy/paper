---
type: concept
aliases: [常数时间恢复, ADR, Accelerated Database Recovery]
---

# Constant Time Recovery

## 定义

SQL Azure 引入的恢复机制（又称 Accelerated Database Recovery），将崩溃恢复时间与事务运行时长解耦，核心手段是把 undo 从「用户事务同步执行」改为异步的 [[Logical Revert|逻辑回滚]]，并把旧版本存储从 TempDB 迁移到用户数据库内的 [[Persistent Version Store]]。

## 核心要点

1. undo 在逻辑上是即时且异步的：版本清理器负责后续物理移除旧版本（逻辑回滚），而非阻塞用户事务或恢复流程。
2. 该设计减少了崩溃恢复期间的不可用时间，也避免了长时间持锁等待 undo 完成。
3. 是 SQL Azure 默认开启的特性，[[Persistent Version Store]] 因此成为用户数据库的一部分而非临时数据库。
4. `PVSPageTrackerList` 在崩溃后可通过重新扫描 PVS 表重建，恢复期间保守地把所有页归入中止事务列表。

## 代表工作

- [[AzureMVCCVersionCleaner]]：在该恢复模型之上进一步优化版本清理的可扩展性与水位推进。

## 相关概念

- [[Persistent Version Store]]
- [[Logical Revert]]
- [[MVCC]]
