---
type: concept
aliases: [Milvus]
---

# Milvus

## 定义

Milvus 是一个从零开始设计的专用向量数据库，专注于高效管理和查询向量数据。采用 LSM-tree 风格的段管理架构，每个段包含独立索引（如 IVFPQ、HNSW）。

## 核心要点

1. 段管理：数据组织为段，每段独立索引，查询广播到所有段后合并
2. 数据新鲜度低：新插入向量需等段合并才能检索
3. 段合并导致 I/O/CPU 峰值，延迟不可预测
4. 支持 DiskANN 磁盘索引

## 代表工作

- [[LindormVector]]: 对比基线，指出 Milvus 段管理架构的延迟和新鲜度问题

## 相关概念

- [[ANNS]]
- [[IVFPQ]]
- [[HNSW]]
- [[DiskANN]]
