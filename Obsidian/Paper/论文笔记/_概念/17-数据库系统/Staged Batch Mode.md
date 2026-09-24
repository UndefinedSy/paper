---
type: concept
aliases: [分阶段批处理模式, SBM]
---

# Staged Batch Mode

## 定义

ByteHouse 面向长时间 ETL 和吞吐优先任务的 stage-based 可恢复执行模式。

## 核心要点

1. exchange boundary 将计划切为 stage。
2. 中间结果物化为轻量 checkpoint。
3. 失败时 task-level retry，且可弹性提高 partition 并行度。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Analytic Pipeline Mode]]
- [[Shared-Storage Architecture]]
