---
type: concept
aliases: [K-Means, K均值聚类, Lloyd's algorithm]
---

# K-Means Clustering

## 定义

K-Means 是一种经典聚类算法，将 $n$ 个数据点分为 $k$ 个簇，迭代执行两步：(1) 质心更新（计算各簇均值），(2) 分配步骤（每个点分配到最近质心），直到收敛。

## 数学形式

$$
\min_{\{C_1, \ldots, C_k\}} \sum_{i=1}^{k} \sum_{\mathbf{x} \in C_i} \|\mathbf{x} - \boldsymbol{\mu}_i\|^2
$$

每轮迭代复杂度 $O(m \cdot d \times k)$（暴力法）或 $O(m \log k)$（ANN 加速）。

## 核心要点

1. 暴力法在高维 + 大 $k$ 时不可行（$k$ 达数十万至百万）
2. LindormVector 用 [[HNSW]] 替代暴力分配，复杂度从 $O(m \times k)$ 降至 $O(m \log k)$
3. 跨迭代重用 HNSW 图（质心仅微移），增量维护降低重建开销
4. 传统 IVF 设 $k = \sqrt{n}$，LindormVector 发现最优值远大于此

## 代表工作

- [[LindormVector]]: ANN 加速 K-Means 支持百万级聚类，5-10x 加速且质量无损
- Spalding-Jamieson et al. (2025): Seeded ANNS K-Means

## 相关概念

- [[IVFPQ]]
- [[HNSW]]
- [[ANNS]]
