---
type: concept
aliases: [Conflict-free Replicated Data Type, 无冲突复制数据类型]
---

# CRDT

## 定义

Conflict-free Replicated Data Type，一种允许多个 replica 独立接受更新并最终收敛到一致状态的数据类型，无需协调即可保证最终一致性。

## 核心要点

1. 支持可交换操作（如计数器增减）时最简单高效
2. 不可交换操作（如删除/清除计数器）需要额外机制
3. [[Bigtable]] 使用 changelog 结构在 [[LSM-tree]] 各层存储 subtotal 和未解决操作
4. Resolved subtotal：不会与其他层 sequencer 重叠的操作的已结算小计
5. Unresolved operations：可能重叠的操作，包括 "future" 最近更新和 "past" 迟到到达
6. Merge compaction 时合并 changelog，修剪已 resolved 的操作

## 代表工作

- [[Bigtable]]: 支持计数器和聚合类型的 CRDT 实现
- [[Cassandra]]: 计数器支持但有删除限制
- [[Redis]]: 轻量级计数器但删除处理复杂

## 相关概念

- [[Eventual Consistency]]
- [[Bigtable]]
- [[LSM-tree]]
