---
type: concept
aliases: [Qdrant]
---

# Qdrant

## 定义

Qdrant 是一个专用向量数据库，仅提供 HNSW 单一密集向量索引，支持磁盘模式。

## 核心要点

1. 仅 HNSW 索引（无 IVFPQ 等替代方案）
2. 大规模构建能力有限：十亿规模 24h 内无法完成索引构建
3. 配置：shard=1, m=32, ef_construct=400

## 代表工作

- [[LindormVector]]: 对比基线之一

## 相关概念

- [[HNSW]]
- [[ANNS]]
