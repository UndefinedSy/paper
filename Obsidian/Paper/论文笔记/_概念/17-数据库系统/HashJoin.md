---
type: concept
aliases: [哈希连接算子]
---

# HashJoin

## 定义

基于哈希表的 Join 执行算子，通常包含 build（构建内表哈希表）与 probe（探测外表匹配）两阶段。

## 核心要点

1. [[TDSQL]] 的向量化实现在 build 阶段对内表 tuple 批量哈希构表；probe 阶段对外表 tuple 批量处理，匹配位置记录进 bitmap 实现无分支批量输出。
2. SIMD 指令加速哈希与键比较；并行哈希表构建与探测利用任务级并行降低单 tuple 处理开销。
3. 是 [[Virtual DN Parallelism]] 重点解决的算子类型之一（窗口聚合、full join 等在随机并行下易冲突）。

## 代表工作

- [[TDSQL]]：build/probe 两阶段均向量化，是全链路向量化引擎的核心算子。

## 相关概念

- [[HashAgg]]
- [[Vectorized Execution Engine]]
- [[SIMD]]
- [[Virtual DN Parallelism]]
