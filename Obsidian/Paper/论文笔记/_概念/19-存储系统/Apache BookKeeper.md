---
type: concept
aliases: [BookKeeper]
---

# Apache BookKeeper

## 定义

一个高性能、低延迟的分布式日志存储服务，提供仅追加（append-only）的 Ledger（账本）抽象，支持 quorum 写入、fencing 机制和非 POSIX API，常被用作分布式系统的 WAL（预写日志）底层实现。

## 核心要点

1. 核心抽象是 Ledger（账本），由多个 Entry 组成，仅支持追加写，一旦关闭不可再修改
2. 通过 ensemble（副本节点集合）+ 写 quorum/确认 quorum/读 quorum 的参数组合，在可用性与一致性之间灵活权衡
3. fencing 机制可在故障转移时安全阻止旧写者继续写入，防止脑裂
4. 被 Apache Pulsar 等消息系统广泛用作底层日志存储

## 代表工作

- [[SalesforceDB]]：[[SDBStore]] 是 Apache BookKeeper 的 fork，扩展出 Extent/Fragment 抽象（对应 Ledger/Entry），并实现了数据缓存与日志缓存两种特化缓存

## 相关概念

- [[SDBStore]]
- [[ZooKeeper]]
- [[Immutability]]
