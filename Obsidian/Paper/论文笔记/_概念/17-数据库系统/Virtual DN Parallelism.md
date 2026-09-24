---
type: concept
aliases: [虚拟数据节点并行]
---

# Virtual DN Parallelism

## 定义

在单个物理 Datanode 内部软件模拟多个虚拟 Datanode（VDN），每个 VDN 独占一段数据范围并自治执行算子，从而实现节点内的无锁、免合并并行执行策略。

## 核心要点

1. 解决传统随机并行（tuple 随机分配给 worker、缺乏全局一致分区）在窗口聚合、full join 等算子上的哈希桶冲突与协调开销问题。
2. 依赖 [[RemoteSubplan]] 算子作为节点内分布边界，按下游分区需求重分布 tuple，保证跨 VDN 数据独立性。
3. 采用混合策略：仅当算子上游存在提供 key 分区输入的 RemoteSubplan 时启用，否则回退随机并行。
4. RemoteSubplan 基于数据分布统计做 range-based 重分区，缓解数据倾斜。

## 代表工作

- [[TDSQL]]：提出该策略实现算子级并行，并与随机并行组成混合策略。

## 相关概念

- [[RemoteSubplan]]
- [[MPP Architecture]]
- [[HashJoin]]
- [[HashAgg]]
