---
type: concept
aliases: [WAL, Write-Ahead Log, 预写日志]
---

# WAL (Write-Ahead Log)

## 定义

预写日志（WAL）是一种数据库持久化机制，在修改数据前先将变更记录写入日志文件，确保故障后可恢复。

## 核心要点

1. LindormTable 和 LindormVector 各自维护 WAL
2. 更新仅在两者 WAL 均持久化后才确认提交
3. 保证跨 LindormTable 数据和 LindormVector 索引的一致性和持久性
4. 支持实时索引更新

## 代表工作

- [[LindormVector]]: 双 WAL 保证数据和索引一致性

## 相关概念

- [[LindormTable]]
- [[LindormVector]]
