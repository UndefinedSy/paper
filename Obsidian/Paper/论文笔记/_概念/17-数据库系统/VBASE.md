---
type: concept
aliases: [VBASE]
---

# VBASE

## 定义

一种统一在线向量相似度搜索与关系查询的系统，通过**放松单调性假设（relaxed monotonicity）**让 ANN 搜索与标量谓词评估交织进行。

## 核心要点

1. **核心思想**：传统 ANN 索引假设候选距离单调递增，VBASE 放松该假设，使向量搜索可以边遍历边评估标量谓词，而不是严格分阶段执行。
2. **交织执行**：把 [[Filtered Search|过滤检索]]从"先召回再过滤"或"先过滤再召回"的二选一，变成图遍历过程中的增量判定。
3. **适用场景**：范围/标签过滤的向量检索，尤其是图索引（如 [[HNSW]]）天然支持的增量遍历。

## 代表工作

- 原始提出于 OSDI 2023（Zhang et al.）。
- [[Nova]]: 其 [[Iterative Query|迭代查询]]机制受 VBASE 启发，利用图索引的增量遍历特性动态扩张搜索前沿。

## 相关概念

- [[Iterative Query]]
- [[Filtered Search]]
- [[HNSW]]
- [[Hybrid Search]]
