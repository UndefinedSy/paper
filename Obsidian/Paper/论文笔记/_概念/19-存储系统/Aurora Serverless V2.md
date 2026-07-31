---
type: concept
aliases: [Aurora Serverless, ASV2]
---
# Aurora Serverless V2
## 定义
Aurora 的 serverless 形态：按 [[Aurora Capacity Unit|ACU]] 为粒度对实例做细粒度、快速的原地垂直伸缩，无需切换实例规格。
## 核心要点
1. 以 ACU 为计量与调容单位，可在秒级尺度上原地增减容量。
2. **非对称伸缩速率**：scale-up 在负载尖峰时快速触发以保证响应性；scale-down 保守，须低利用率持续较久才触发，以防资源抖动（thrashing）。
3. [[Aurora Limitless Database]] 的垂直伸缩完全建立在它之上，并叠加了 [[Proportional ACU Allocation]] 来在 shard group 成员间分配配额。
4. Limitless 还为 shard 与 router 使用**不同的伸缩信号与内存策略**：shard 看 compute 并多给 buffer cache，router 看 memory 并多给 heap。
5. 同时也是控制平面 provision/deprovision 工作流的基础设施。
## 代表工作
- Resource management in Aurora Serverless (VLDB 2024)。
- [[AuroraLimitless]]：垂直伸缩的直接依赖。
## 相关概念
- [[Aurora Capacity Unit]]
- [[Proportional ACU Allocation]]
- [[Amazon Aurora]]
- [[Resource Pool]]
