---
type: concept
aliases: [Sorted String Table, SST]
---

# SSTable

## 定义

Sorted String Table，一种不可变的、按 key 排序的键值存储文件格式，是 [[LSM-tree]] 的磁盘持久化形式。

## 核心要点

1. 不可变文件格式，key 有序排列
2. 由 minor compaction 从 memtable 生成
3. 可通过 [[Bloom Filter]] 快速判断某 key 是否存在
4. 在 [[Bigtable]] 中，每个 [[Locality Group|locality group]] 在 compaction 时生成独立 SSTable
5. 存储在分布式文件系统 [[Colossus]] 上，tablet 移动不需要数据迁移

## 代表工作

- [[Bigtable]]: 核心持久化格式
- [[HBase]]: 使用 SSTable 的 HFile 变体
- [[Cassandra]]: 使用 SSTable 格式

## 相关概念

- [[LSM-tree]]
- [[Bloom Filter]]
- [[Colossus]]
- [[Bigtable]]
