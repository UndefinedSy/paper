---
type: concept
aliases: [Approximate Nearest Neighbor Search, ANNS, ANN, 向量搜索, 语义搜索]
---

# ANNS (Approximate Nearest Neighbor Search)

## 定义

近似最近邻搜索（ANNS）是在高维向量数据集中高效检索与查询向量最相似的 $k$ 个向量的技术，牺牲少量精度换取大幅延迟降低。

## 核心要点

1. 动机：精确 NN 搜索在高维空间受维度诅咒影响，复杂度指数级
2. 评估指标：recall@k = 检索到的 $k$ 个中包含真实最近邻的比例
3. 主要方法族：
   - 量化方法：[[Product Quantization]], [[RaBitQ]]
   - 图方法：[[HNSW]], NSG, [[DiskANN]]
   - 倒排方法：[[IVFPQ]]
4. LindormVector 的 ANN 应用：加速 K-Means 聚类 + 加速质心搜索

## 代表工作

- [[LindormVector]]: 分布式 IVFPQ + ANN 加速 K-Means
- [[Milvus]]: 专用向量数据库
- [[DiskANN]]: 磁盘向量索引

## 相关概念

- [[IVFPQ]]
- [[HNSW]]
- [[Product Quantization]]
- [[DiskANN]]
