---
title: "Vector Sharding"
category: "数据库系统"
tags: [distributed-vector-search, sharding, routing]
created: 2026-08-07
---
# Vector Sharding

[[Vector Sharding]] 将向量集合按相似度或空间区域切分到多个节点，使每个节点仅维护局部索引；查询路由到可能包含近邻的分片后合并结果。

## 在 GaussDB-Vector 中

[[GaussDBVector]] 按到聚类中心的距离将向量分配给 DN。CN 保留中心和基数模型，按范围/top-$k$/标量选择率决定查询 fan-out；检测到超过阈值的分布漂移后触发后台重分布。

## 关联
- [[GaussDBVector]]
- [[Sharding]]
- [[Approximate Nearest Neighbor Search]]
