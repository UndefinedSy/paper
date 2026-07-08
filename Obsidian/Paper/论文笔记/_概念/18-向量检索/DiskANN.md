---
type: concept
aliases: [DiskANN, Vamana]
---

# DiskANN

## 定义

DiskANN 是一种面向大规模磁盘驻留数据集的近似最近邻索引，基于近邻图，通过放松严格连通性约束、引入磁盘感知布局、向量重排序和预取来减少图遍历中的随机 SSD I/O。

## 核心要点

1. 磁盘优化：减少随机 I/O，支持单节点十亿级向量搜索
2. 放松连通性：比 HNSW/NSG 更宽松的图连通约束
3. 向量重排序：将相邻图节点的向量连续存储，提高预取效率
4. 局限：图遍历需反复获取全精度向量（多轮 get），更新处理差

## 代表工作

- [[LindormVector]]: 对比指出 DiskANN 在共享存储架构中 I/O 开销过大，选择 IVFPQ 代替
- [[Milvus]]: 使用 DiskANN 作为磁盘索引
- [[CosmosDB]]: 采用 DiskANN 算法

## 相关概念

- [[ANNS]]
- [[HNSW]]
- [[IVFPQ]]
