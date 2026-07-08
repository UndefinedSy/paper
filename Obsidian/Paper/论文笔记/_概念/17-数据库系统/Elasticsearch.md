---
type: concept
aliases: [Elasticsearch, ES]
---

# Elasticsearch

## 定义

Elasticsearch 是一个分布式搜索和分析引擎，通过插件扩展支持向量搜索。采用段管理架构，最近引入 DiskBBQ 磁盘导向索引。

## 核心要点

1. 通用搜索引擎 + 向量搜索扩展
2. DiskBBQ：磁盘导向的二值量化索引
3. 段管理架构的固有局限（查询广播、段合并开销）
4. 在十亿规模评估中构建时间较长

## 代表工作

- [[LindormVector]]: 对比基线之一，在相同召回率下 LindormVector QPS 领先 1-2 个数量级

## 相关概念

- [[ANNS]]
- [[Milvus]]
