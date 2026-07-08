---
type: concept
aliases: [Bigtable Coprocessor, 协处理器]
---

# Coprocessor

## 定义

Bigtable 的内部执行层，允许在 tablet server 上直接运行逻辑，支持将计算推送到数据所在位置（compute-to-data）。

## 核心要点

1. 允许在 tablet server 上执行用户定义的逻辑
2. 用于物化视图的 server-side MapReduce 流水线
3. Mapper coprocessor 处理源表更新并传播到 hidden shuffle table
4. Reducer coprocessor 聚合 shuffle table 中的分片
5. Finalizer coprocessor 将结果编码到输出表
6. 源自 Jeffrey Dean 在 LADIS'09 的分布式系统设计分享

## 代表工作

- [[Bigtable]]: 物化视图的执行引擎

## 相关概念

- [[Bigtable]]
- [[MapReduce]]
- [[Materialized Views]]
