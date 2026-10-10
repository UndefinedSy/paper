---
type: concept
aliases: [XCN]
---

# Transaction Commit Number

## 定义

SalesforceDB 中用于编码读、提交时间戳的逻辑时钟，缩写 XCN，实现为 64 位无符号整数，近似表示纳秒级系统时间。

## 核心要点

1. [[Coordinator]] 维护两个 XCN：**commit epoch**（当前提交 epoch，用于给提交事务分配时间戳）与 **visible epoch**（当前可见 epoch，用于给以 [[Snapshot Isolation]] 运行的语句分配读快照时间戳）
2. 保证 commit epoch 单调递增且严格大于 visible epoch
3. Epoch Coordinator 大约每 1ms 推进一次，推进时 visible epoch 更新为不小于上一 epoch 内分配的最大提交时间戳
4. [[Executor]] 使用最近接收到的 visible epoch 在本地分配快照时间戳，无需每次 RPC
5. 外部一致性依赖它：提交返回前，Executor 需确认本地 visible epoch 已越过该事务的 commit XCN 并再推进一次，保证集群任意节点后续查询均可见该提交

## 代表工作

- [[SalesforceDB]]：XCN 是其外部一致性与 MVCC 快照隔离机制的核心时间戳抽象，类似 Spanner 的 TrueTime 但不依赖硬件时钟同步

## 相关概念

- [[Coordinator]]
- [[MVCC]]
- [[Snapshot Isolation]]
- [[Spanner]]
