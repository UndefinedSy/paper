---
type: concept
aliases: [Apache HBase]
---

# HBase

## 定义

开源的非关系型数据存储系统，以 [[Bigtable]] 为蓝本设计，使用 Java 实现，API 和核心功能与 Bigtable 高度相似。

## 核心要点

1. 开源 Java 实现，直接 modeled after [[Bigtable]]
2. API 和核心功能与 Bigtable 高度相似
3. 使用 [[LSM-tree]] 存储引擎，HFile 格式对应 [[SSTable]]
4. 被多家知名企业采用，是 Bigtable 影响力的直接体现
5. 运行在 HDFS 上（对应 Bigtable 的 [[Colossus]]）
6. 是 NoSQL 运动的重要代表

## 代表工作

- HBase: 开源 Bigtable 实现

## 相关概念

- [[Bigtable]]
- [[Cassandra]]
- [[LSM-tree]]
- [[SSTable]]
