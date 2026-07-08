---
type: concept
aliases: [WAL, Write-Ahead Log, 预写日志]
---

# WAL (Write-Ahead Log)

## 定义

预写日志（WAL）是一种数据库持久化机制，在修改数据前先将变更记录写入日志文件，确保故障后可恢复。

## 核心要点

1. 在修改数据前先将变更记录写入日志文件，确保故障后可恢复
2. [[Bigtable]] 中称为 commit log，所有 mutation 先写 WAL 再确认给调用者
3. LindormTable 和 LindormVector 各自维护 WAL，更新仅在两者 WAL 均持久化后才确认提交
4. Commit log 支持分片以维持/提升写吞吐（单文件吞吐有限）
5. 只读 tablet 在无更新后释放 commit log 引用，减少恢复时间和资源

## 代表工作

- [[Bigtable]]: WAL (commit log) + [[LSM-tree]] 存储引擎
- [[LindormVector]]: 双 WAL 保证数据和索引一致性

## 相关概念

- [[Bigtable]]
- [[LSM-tree]]
- [[LindormTable]]
- [[LindormVector]]
- [[SSTable]]
- [[CRC32]]
