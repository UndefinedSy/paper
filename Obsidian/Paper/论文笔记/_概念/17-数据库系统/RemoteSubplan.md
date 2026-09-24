---
type: concept
aliases: [远程子计划算子]
---

# RemoteSubplan

## 定义

分布式查询计划中负责跨节点/跨虚拟节点重分布数据（数据 shuffle）的算子，是分布边界的核心实现单元。

## 核心要点

1. 按下游算子的分区需求（如 Join key）重新分布上游 tuple，保证下游各分区数据独立、结果可正确聚合。
2. 是 [[Virtual DN Parallelism]] 判定是否启用的关键信号：只有上游存在提供 key 分区输入的 RemoteSubplan 时才启用 VDN 并行。
3. 可基于数据分布统计做 range-based 重分区，用于缓解数据倾斜。

## 代表工作

- [[TDSQL]]：在 [[Virtual DN Parallelism]] 与并行 Fragment 执行中作为核心数据重分布算子。

## 相关概念

- [[Virtual DN Parallelism]]
- [[MPP Architecture]]
