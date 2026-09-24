---
type: concept
aliases: [Massively Parallel Processing, 大规模并行处理架构]
---

# MPP Architecture

## 定义

将数据与计算切分到多个独立节点、各节点并行处理本地数据分片、通过网络交换中间结果完成分布式查询的数据库架构范式。

## 核心要点

1. 典型代表包括 [[Greenplum]]、Vertica、Teradata，早期以列存 + 批量 ETL 为主。
2. 节点间数据重分布（Join、聚合）通常需要建立点对点网络连接，集群规模增大时连接数呈 $O(N^2)$ 增长，即"连接爆炸"问题。
3. [[TDSQL]] 在传统 MPP 基础上引入 [[Forward Node]] 层，把物理全连接拓扑收敛为星型拓扑，兼顾低延迟与可扩展性。

## 代表工作

- [[Greenplum]]：经典开源 MPP 数据库。
- [[TDSQL]]：在原生 MPP 框架上做 Forward Node、[[Virtual DN Parallelism]]、向量化与 Runtime Filter 的协同优化。

## 相关概念

- [[Forward Node]]
- [[Shared Memory]]
