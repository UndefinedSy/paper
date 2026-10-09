---
type: concept
aliases: [free node array, 空闲节点数组, free-slot array]
---

# Free Node Array

## 定义

图索引中用于记录已回收、可供复用的节点槽位的数据结构，由 `vacuum` 维护、由 `insert` 消费。

## 核心要点

1. **回收来源**：`ctid` 列表为空、或写入被中止的过时节点。
2. **插入时复用**：插入先从数组**尾部**取一个空闲槽 ID；若成功，则复用该节点原有的层数而不重新随机生成，避免层数分布的额外扰动。
3. **归还**：如果这次写入最终通过追加到已有 `ctid` 列表完成（即所取节点因[[Duplicate Vector Merging|重复向量合并]]而未被使用），空闲节点被归还数组。
4. **线程安全**：索引扫描**从不访问**该数组，且 Nova 强制 `flush` 与 `vacuum` 互斥，因此对数组的操作天然安全。
5. **与 `mark_free` 的配合**：`mark_free` **只把槽位加入空闲数组、不清空节点内容**，避免并发查询读到被清空的节点；同时用内存屏障实现 release-acquire 语义。

## 代表工作

- [[Nova]]: [[Nova Memory|NovaM]] 空间复用的核心结构，用于抑制图索引的空间膨胀。

## 相关概念

- [[Nova Memory]]
- [[Duplicate Vector Merging]]
- [[HNSW]]
