---
type: concept
aliases: [InnoDB Purge, MySQL Purge Thread]
---

# InnoDB Purge

## 定义

MySQL InnoDB 存储引擎中持续运行的后台 purge 任务，从 undo 日志中增量移除不再需要的旧版本数据。

## 核心要点

1. InnoDB 旧版本存放在数据行之外的 undo 日志中（而非行内），只维护 diff 而非完整行拷贝，空间效率高。
2. Purge 是增量、持续、与事务处理交织进行的，而非大周期性 sweep；依赖元数据直接定位待清理的 undo 条目，无需扫描整表。
3. 关键差异：InnoDB 的回滚（rollback）由用户事务同步执行，purge 线程不负责回滚；SQL Azure 的 undo 是逻辑上即时、异步执行的，物理移除交给版本清理器（[[Logical Revert]]）。
4. 与 Autovacuum 一样，若有事务仍需要 undo 日志中的旧版本，清理会被阻塞。

## 代表工作

- [[AzureMVCCVersionCleaner]]：Related Work 中与之对比版本存储位置与清理时机的差异。

## 相关概念

- [[MVCC]]
- [[Logical Revert]]
- [[Persistent Version Store]]
