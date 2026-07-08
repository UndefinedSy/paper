---
type: concept
aliases: [Apache Cassandra]
---

# Cassandra

## 定义

由 Facebook 开发的开源去中心化非关系型数据库系统，数据模型类似 [[Bigtable]] 但支持嵌套 column family 和 CQL 查询语言。

## 核心要点

1. 去中心化架构，无单点故障
2. 数据模型类似 [[Bigtable]]，但 column family 可嵌套
3. 支持 Cassandra Query Language (CQL)
4. 使用 [[LSM-tree]] 存储引擎和 [[SSTable]] 格式
5. 计数器支持但有删除技术限制
6. 受 [[Bigtable]] 启发，是 NoSQL 运动的代表系统之一

## 相关概念

- [[Bigtable]]
- [[HBase]]
- [[DynamoDB]]
- [[LSM-tree]]
- [[SSTable]]
