---
type: concept
aliases: [恢复点目标, Recovery Point Objective]
---
# RPO

## 定义
Recovery Point Objective（RPO）是容灾指标：系统发生故障时，**可容忍丢失的数据量**，通常以时间衡量——即「最多会丢多久的数据」。

## 核心要点
1. **与 RTO 的区别**：RTO（Recovery Time Objective）衡量**恢复要多久**，RPO 衡量**要丢多少**。二者常一起作为 SLA 条款。
2. **由复制模式决定**：
   - 同步复制（[[K2Plus]] 同步模式）→ RPO ≈ 0，但事务关键路径要等跨区域确认（实测吞吐最多降 42%、延迟约 +45 ms）；
   - 异步复制 → RPO > 0，但写入路径不受跨区域 RTT 拖累。
3. **[[K2Plus]] 的承诺**：异步复制模式下区域故障时**保证 RPO ≤ 10 s**，并通过 epoch 协商把数据回滚到所有受影响分区都已完整复制的最新 [[Epoch]]。
4. **工程含义**：RPO 是业务分级配置的依据——[[K2Plus]] 允许**按表或 global key range**独立选择同步/异步模式，让 mission-critical 表付同步的代价、非关键表享受异步的吞吐。

## 代表工作
- [[K2Plus]]：同步/异步复制可按表配置，异步模式区域故障 RPO ≤ 10 s。

## 相关概念
- [[Replication Middleware]]
- [[Region Failover]]
- [[Epoch]]
- [[SLO]]
