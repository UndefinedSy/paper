---
type: concept
aliases: [Materialized View, MV, 物化视图]
---

# Materialized Views

## 定义

预计算并持久化存储的查询结果视图，在源数据更新时持续刷新，避免每次查询时重新计算复杂转换。

## 核心要点

1. 用户定义标准 [[GoogleSQL]] 查询作为转换
2. [[Bigtable]] 在数据到达时持续执行，利用 [[Coprocessor]] 框架
3. 执行流程：源表更新 → mapper coprocessor → hidden shuffle table → reducer coprocessor → 输出表
4. 使用 watermark 跟踪各阶段处理进度，确保最终一致性
5. 替代外部 MapReduce/Flume 流水线，减少架构复杂度和运维开销
6. 每个复制表副本维护自己的物化视图，传播严格在复制之后

## 代表工作

- [[Bigtable]]: 原生物化视图

## 相关概念

- [[Bigtable]]
- [[Coprocessor]]
- [[MapReduce]]
- [[Change Data Capture]]
- [[GoogleSQL]]
