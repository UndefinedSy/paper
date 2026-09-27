---
type: concept
aliases: [区域故障切换, Geo-Region Failover]
---
# Region Failover

## 定义
Region Failover 是当一个地理区域整体（或区域内多数 AZ）不可用时，把该区域承担的数据服务**转移**到其他区域的过程。

## 核心要点
1. **[[K2Plus]] 分类的三类区域故障**：
   - 区域内两个 AZ 的全部 [[DFV]] 实例故障 → 该区域存储池变为只读；
   - 区域内两个 AZ 的 manager 集群（RM + replicator manager）故障 → 管理能力丧失；
   - 区域大规模故障（如 ≥80% 数据节点因断电崩溃）且无法自动恢复 → 强制执行。
2. **恢复区域选择**：选持有该分区 global range 覆盖的**最近区域**，以维持工作负载局部性。
3. **恢复流程**：同步复制模式下按分区迁移语义重建，差别是**先更新 [[Global Manager]] 的全局 range view**，再更新 [[Regional Manager]] 的区域分区位置。
4. **异步复制的 RPO 代价**：部分新写可能未跨区域复制，**不可避免地丢失部分数据，但保证 RPO ≤ 10 s**。恢复难点在于把数据恢复到**既保持事务写原子性、又保持事务全局序**的快照——[[K2Plus]] 借助 [[Epoch]] 分段能力，通过协商选取「在所有受影响分区上均已完整复制的最新 epoch」作为恢复点。
5. **两种恢复模式**：活跃分区直接提升副本为主；非活跃分区需 [[Cold Start]]（额外约 27 s）。

## 代表工作
- [[K2Plus]]：区域故障实验中副本提升立即完成，cold-start 额外约 27 s。

## 相关概念
- [[Cold Start]]
- [[Global Manager]]
- [[Regional Manager]]
- [[Epoch]]
- [[RPO]]
- [[Availability Zone]]
