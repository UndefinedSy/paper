---
type: concept
aliases: [RNG, Relative Neighborhood Graph, 相对邻域图]
---

# RNG (Relative Neighborhood Graph)

## 定义

相对邻域图（RNG）是一种几何图结构，如果两点之间不存在第三点同时与两者都更近，则它们之间有边相连。HNSW 的边选择启发式受 RNG 启发。

## 核心要点

1. RNG 保证图的连通性和可导航性
2. HNSW 使用受 RNG 启发的启发式选择邻居，避免过长的边
3. NSG（Navigable Small World Graph）也使用类似剪枝规则

## 相关概念

- [[HNSW]]
- [[ANNS]]
