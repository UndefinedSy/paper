---
title: "Inverted File Index (IVF)"
category: "数据库系统"
tags: [ann, vector-index, clustering]
created: 2026-08-07
---
# Inverted File Index (IVF)

[[Inverted File Index|IVF]] 先聚类向量并把每个向量归入一个倒排桶；查询只探测靠近查询向量的若干桶，从而减少距离计算和数据访问。

## 在 GaussDB-Vector 中

[[GaussDBVector]] 通过元页、聚类页、数据页和可选 [[Product Quantization|PQ]] 页实现持久化 IVF；同一 cluster 的向量物理连续，删除采用逻辑标记并由 vacuum 重写存活项。

## 关联
- [[Product Quantization]]
- [[Approximate Nearest Neighbor Search]]
- [[GaussDBVector]]
