---
type: concept
aliases: [CDC, 变更数据捕获]
---

# Change Data Capture

## 定义

一种数据库技术，允许用户实时接收对表的所有变更（mutation），用于实时分析、数据同步/复制和欺诈检测。

## 核心要点

1. Stream 在 tablet 边界分片以保证可扩展性
2. 消费者需在 tablet 分裂/合并或复制拓扑变化时管理状态交接
3. 位置跟踪三个维度：行、replica、逻辑时间
4. Continuation token 记录处理位置，支持流的重分片
5. CDC Beam Adapter 运行在 [[Google Cloud Dataflow|Cloud Dataflow]] 上，保证至少一次执行
6. 控制面确保每个 tablet 最多一个 worker 在流式读取

## 代表工作

- [[Bigtable]]: 原生 CDC 支持，驱动物化视图和实时分析

## 相关概念

- [[Bigtable]]
- [[Google Cloud Dataflow]]
- [[Materialized Views]]
