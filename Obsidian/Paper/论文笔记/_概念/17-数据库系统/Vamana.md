---
title: "Vamana"
category: "数据库系统"
tags: [ann, graph-index, vector-database]
created: 2026-08-07
---
# Vamana

[[Vamana]] 是 [[DiskANN]] 使用的有向单调相对邻近图（MRNG）式近似最近邻图。它限制每个节点的最大出度，并通过带松弛系数的边剪枝保留跨区域快捷边，从而在高召回下缩短图搜索路径并控制磁盘 I/O。

## 在 GaussDB-Vector 中

[[GaussDBVector]] 将 Vamana 改造成可持久化、可更新的页式索引：按维度选择节点/PQ code/边的布局；以双簇重叠子图建图；用最远邻居启发式降低增量插入代价；通过逻辑删除和 vacuum 重插修复图连通性。

## 关联
- [[DiskANN]]
- [[Approximate Nearest Neighbor Search]]
- [[Product Quantization]]
- [[GaussDBVector]]
