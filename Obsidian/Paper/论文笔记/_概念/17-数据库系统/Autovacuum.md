---
type: concept
aliases: [Autovacuum, PG Vacuum, PostgreSQL 自动清理]
---

# Autovacuum

## 定义

PostgreSQL 周期性运行的后台清理进程，扫描并回收死元组（dead tuple），是 PostgreSQL 版本对 MVCC 垃圾回收的实现。

## 核心要点

1. 与 SQL Azure Version Cleaner 类似，都周期性运行、利用元数据避免全表扫描、遇冲突负载会推迟清理。
2. 关键差异：PostgreSQL 更新是原地失效旧元组、插入新元组，所有旧版本与最新版本一起存放在主表内，而非独立版本存储；Autovacuum 分两趟——先收集死元组 ID 并清理索引，再重读部分页清理表。
3. 并行粒度不同：Autovacuum 可以多进程并行，但同一时刻单表/索引只有一个进程清理；SQL Azure Version Cleaner 支持多线程并发清理同一大表。
4. PostgreSQL 32 位事务 ID 存在 wraparound 风险，Autovacuum 还需承担防止 wraparound 的职责；SQL Azure 用内部 48 位事务 ID 且仅分配给写事务，风险更低。

## 代表工作

- [[AzureMVCCVersionCleaner]]：Related Work 中与之对比架构差异。

## 相关概念

- [[MVCC]]
- [[PostgreSQL]]
- [[Persistent Version Store]]
