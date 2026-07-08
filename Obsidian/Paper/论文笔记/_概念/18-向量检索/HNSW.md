---
type: concept
aliases: [Hierarchical Navigable Small World, HNSW]
---

# HNSW

## 定义

HNSW（层次化可导航小世界图）是一种基于图的近似最近邻索引。将数据向量组织为节点，跨多层构建图：底层包含所有向量，上层从下层采样形成更稀疏的图。查询从顶层贪心下降到底层。

## 核心要点

1. 多层结构：顶层稀疏，底层密集，实现快速远距离跳转 + 精确细化
2. 每个节点最多 $M$ 条出边，边的选择受 [[RNG]] 启发式指导
3. 索引大小 $O(Mn)$
4. 查询复杂度近似 $O(\log n)$
5. LindormVector 的创新：用 PCA 压缩质心进行图遍历减少内存带宽，再重排

## 代表工作

- [[LindormVector]]: 在质心上构建 HNSW 加速质心搜索；在 K-Means 中用 HNSW 替代暴力分配
- Faiss: 提供 HNSW 实现

## 相关概念

- [[ANNS]]
- [[IVFPQ]]
- [[K-Means Clustering]]
- [[PCA]]
