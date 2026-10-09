---
type: concept
aliases: [covering index, 覆盖索引]
---

# Covering Index

## 定义

把查询中频繁使用的附加属性物化进索引结构内部，使过滤或取值无需回表（访问堆）即可完成的索引设计。

## 核心要点

1. **动机**：真实应用的高频查询常在特定结构化字段上过滤——例如 RAG 场景几乎**所有查询**都需按 document ID 过滤。
2. **向量场景的实现**：把标量属性直接物化在 ANN 索引结构内部（如 [[Nova Memory|NovaM]] 的图节点中），过滤可在索引扫描期间完成。
3. **收益**：免除额外的堆访问（heap access），显著降低 I/O 开销。
4. 思想源自关系数据库的覆盖索引，被迁移到向量索引（尤其是图索引）中。

## 代表工作

- [[Nova]]: 受关系数据库启发将该思想引入向量引擎，与 [[Partial Index]] 共同构成混合查询优化手段。

## 相关概念

- [[Partial Index]]
- [[Hybrid Search]]
- [[Filtered Search]]
- [[Nova Memory]]
