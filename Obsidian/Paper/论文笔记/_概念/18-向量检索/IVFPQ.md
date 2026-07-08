---
type: concept
aliases: [Inverted File with Product Quantization, IVFPQ, IVF-PQ]
---

# IVFPQ

## 定义

IVFPQ 是一种组合向量索引方法，先使用倒排文件（IVF）将向量聚类分区，再在分区内部使用乘积量化（PQ）压缩向量，实现近似最近邻搜索。

## 核心要点

1. IVF 层：使用 K-Means 将 $n$ 个向量分为 $nlist$ 个簇，查询时仅扫描 $nprobe$ 个最近簇
2. PQ 层：将每个向量编码为紧凑的量化码，在内存中快速计算近似距离
3. 最终重排：对 PQ 粗排的 top-k 候选，获取全精度向量计算精确距离
4. 传统启发式设 $nlist = \sqrt{n}$，但 LindormVector 发现最优 $nlist$ 远大于此值

## 代表工作

- [[LindormVector]]: 采用超大 nlist 的 IVFPQ，配合 ANN 加速 K-Means 和优化质心搜索
- Faiss: Facebook 的相似性搜索库，广泛实现 IVFPQ

## 相关概念

- [[Product Quantization]]
- [[K-Means Clustering]]
- [[ANNS]]
- [[HNSW]]
