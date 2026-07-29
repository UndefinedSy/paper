---
type: concept
aliases: [Bw-tree, 无锁 B-tree]
---

# BW-Tree

## 定义

一种面向多核和内存/闪存系统的无锁 B-tree 变体，以不可变页面、delta record 和间接映射表实现并发更新与页面地址稳定。

## 核心要点

1. 更新追加 delta，而不是原地修改基础页。
2. Mapping Table 将逻辑 Page ID 映射到物理地址，便于页面迁移。
3. 周期性 consolidation/compaction 合并过长 delta chain。
4. [[ByteGraph]] 将每个 vertex 映射为一棵 BW-Tree，以 LeafPage 存储邻接表。

## 代表工作

- [[ByteGraph-Dione]]：在 BW-Tree append-only LeafPage 中统一实现 row/column 格式与 MVCC。

## 相关概念

- [[ByteGraph]]
- [[Adjacency List]]
- [[Adaptive Dual-Format Graph Storage]]
