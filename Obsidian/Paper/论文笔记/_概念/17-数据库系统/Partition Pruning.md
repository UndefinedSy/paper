---
type: concept
aliases: [分区裁剪, Partition Elimination]
---
# Partition Pruning
## 定义
PostgreSQL 的优化机制：依据查询条件在规划或执行期排除不可能包含结果的分区，从而只扫描必要分区。
## 核心要点
1. 静态裁剪在规划期完成（条件为常量），动态裁剪在执行期完成（条件依赖参数或 join 结果）。
2. [[Aurora Limitless Database]] 借它判断某查询所需的源 tuple 是否**全部位于同一个 shard**。
3. 若是，则该 **single-shard 查询**被整体下推到该 shard，router↔shard 往返更少、延迟更低。
4. 🎯 **单分片查询是系统的 sweet spot**：选对 shard key、恰当标记 distributed 函数可提高单分片查询占比，从而改善整体性能。
5. 在 Limitless 的两层分区结构中，router 层与 shard 层都能各自受益于裁剪。
## 代表工作
- [[AuroraLimitless]]：用它识别并下推单分片查询。
- [[PostgreSQL]]：机制提供者。
## 相关概念
- [[PostgreSQL]]
- [[Sharding]]
- [[Foreign Data Wrapper]]
- [[Table Slice]]
