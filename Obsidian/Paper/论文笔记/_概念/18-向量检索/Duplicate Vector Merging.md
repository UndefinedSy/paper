---
type: concept
aliases: [duplicate vector merging, 重复向量合并]
---

# Duplicate Vector Merging

## 定义

允许图索引中的单个节点关联多个 `ctid`，从而避免为语义重复的向量重复创建物理节点的空间优化。

## 核心要点

1. **动机**：原版 [[HNSW]] 为每个重复向量创建独立节点，导致**索引膨胀与 recall 退化**。
2. **两个重复来源**：（a）应用层显式插入相同向量；（b）[[MVCC]] 下非 **HOT（Heap Only Tuple）** 更新——PostgreSQL 创建带新 `ctid` 的新堆元组并把写入传播到所有索引（含 ANN 索引），产生携带相同向量但 `ctid` 不同的新索引项。此外，flush 事务在 rollback 后重试也会产生重复。
3. **做法**：插入时检查 layer 0 是否有候选邻居落在预定义距离阈值内；若是且其 `ctid` 列表未满，则以**无锁方式**把新 `ctid` 追加到该列表并完成写入；否则在 base file 中创建新的物理节点。
4. **与空间复用的交互**：若插入最终通过追加完成（所取空闲槽未被使用），该空闲节点被归还给空闲数组。

## 代表工作

- [[Nova]]: [[Nova Memory|NovaM]] 的核心工程优化之一，直接针对关系数据库 MVCC 与图索引结合的写放大问题。

## 相关概念

- [[Nova Memory]]
- [[HNSW]]
- [[MVCC]]
- [[Free Node Array]]
