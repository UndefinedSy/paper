---
type: concept
aliases: [PGVector, PostgreSQL Vector]
---

# PGVector

## 定义

PGVector 是 PostgreSQL 的向量搜索扩展，支持 HNSW 和 IVFFLAT 两种索引类型。

## 核心要点

1. 构建于关系数据库之上，提供完整 ACID 保证
2. ACID 开销（锁、事务验证）增加查询延迟
3. HNSW 索引构建在 PostgreSQL 上极慢（每向量插入触发随机 I/O）
4. 大规模 $nlist$ 导致 OOM，只能用极小 $nlist=10K$

## 代表工作

- [[LindormVector]]: 对比基线，指出 ACID 开销对向量搜索不必要

## 相关概念

- [[IVFPQ]]
- [[HNSW]]
- [[ANNS]]
