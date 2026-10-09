---
type: concept
aliases: [bitmap filtering, 位图过滤, bitmap pre-filtering]
---

# Bitmap Filtering

## 定义

先根据标量谓词构造满足条件的行 ID 位图，再在向量索引扫描时用该位图快速跳过不合格向量的预过滤技术。

## 核心要点

1. **位置**：对应 [[Nova]] 混合查询计划中的 **Type 2（Plan B/C）**——`scalar index scan` → `bitmap` → `vector index scan`。
2. **语义**：预过滤。只有满足标量谓词的向量才被允许进入索引扫描。
3. **与回调式预过滤的区别**：Type 2 **提前物化**显式位图；Type 4（On-the-fly Filtering）把谓词表达为**回调函数**，在索引遍历中逐候选 `ctid` 求值，从而避免位图构造开销。两者语义相同、物化方式不同。
4. **与后过滤的区别**：后过滤（Type 3）先从索引取 $\sigma \times k$ 个候选再验证谓词，位图过滤则在扫描阶段就排除不合格向量。
5. **代价考量**：位图构造本身有开销（写入、内存占用），在高选择性查询中可能不划算，因此需要代价模型决定是否采用。

## 代表工作

- [[Nova]]: 在四类混合查询计划中作为 Type 2；其 Type 4 变体正是为规避位图构造代价而设计。

## 相关概念

- [[Hybrid Search]]
- [[Filtered Search]]
- [[Partial Index]]
