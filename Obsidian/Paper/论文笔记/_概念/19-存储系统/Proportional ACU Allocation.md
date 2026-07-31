---
type: concept
aliases: [按比例 ACU 分配, 动态容量配额]
---
# Proportional ACU Allocation
## 定义
[[Aurora Limitless Database]] 的容量分配策略：把 shard group 的总 min/max [[Aurora Capacity Unit|ACU]] 预算，按各节点实际消耗量的占比切分为每节点的动态配额。
## 核心要点
1. 公式为 $\text{dynamicMin/MaxACU}_i = \text{shardGroupMin/MaxACU} \cdot \frac{\text{consumedACU}_i}{\sum_j \text{consumedACU}_j}$。
2. 初始时预算在全体成员间**均分**，之后周期性地按实际消耗重新分配。
3. 设计依据：消耗更多容量的节点通常有更高的增长需求，需要更快扩容，因此应获得更大的配额上限。
4. 新增节点时，其预算从**未被消耗的容量**中划出；余量不足则直接拒绝客户的扩容请求。
5. ⚠️ 潜在问题：该策略会强化既有热点节点的配额优势，但并不主动纠正数据分布不均——纠正不均要靠 shard split（[[Table Slice]] 迁移）。
## 代表工作
- [[AuroraLimitless]]：本策略的提出与实现。
## 相关概念
- [[Aurora Capacity Unit]]
- [[Aurora Serverless V2]]
- [[Table Slice]]
- [[Resource Pool]]
