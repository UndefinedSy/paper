---
type: concept
aliases: [LSM Tree, Log-Structured Merge-tree, LSM tree, LSM-Tree]
---

# LSM-tree

## 定义

一种为写优化设计的数据结构，将数据按层次组织，新数据先写入内存缓冲区（memtable），满后刷入磁盘形成不可变文件（SSTable），多层累积后通过压缩合并。

## 核心要点

1. 数据先写入内存中的 [[WAL|memtable]]，保证持久性后写入 [[WAL]]
2. Memtable 满后通过 minor compaction 刷入磁盘形成 SSTable
3. 多层 SSTable 累积过深时触发 merge compaction 合并
4. Major compaction 合并所有 SSTable 为一个，执行删除和垃圾回收
5. 读操作需要遍历多层 SSTable，读放大是主要代价

## 代表工作

- [[Bigtable]]: 使用 LSM-tree + WAL 作为核心存储引擎
- [[Cassandra]]: 使用 LSM-tree 结构存储数据
- [[HBase]]: 基于 Bigtable 的 LSM-tree 实现

## 相关概念

- [[SSTable]]
- [[WAL]]
- [[Bloom Filter]]
- [[Bigtable]]
