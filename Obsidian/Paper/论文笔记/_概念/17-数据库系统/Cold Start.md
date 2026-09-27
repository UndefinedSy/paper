---
type: concept
aliases: [冷启动恢复, Cold-Start Recovery]
---
# Cold Start

## 定义
在 [[K2Plus]] 的语境中，Cold Start（冷启动恢复）指区域故障后对**在备区域未处于活跃状态的分区**进行恢复：需要**重建内存组件（memtable、索引）并向管理面注册**数据分区，而不只是提升一个已在运行的副本。

## 核心要点
1. **两种区域恢复模式**：
   - **分区由数据节点在服务** → 只需在 [[Global Manager]] / [[Regional Manager]] 更新元数据，把副本提升为主（立即完成）；
   - **分区处于非活跃状态** → 需 cold-start recovery，额外约 **27 s**。
2. **为什么会有非活跃分区**：这是资源效率的取舍——让所有分区在所有区域都保持 warm 代价过高，因此 K2+ 采用「active-on-demand」策略，平时只在副本区域保留存储层的复制（靠 [[DFV]] + replicator 保证可靠），不预先运行服务实例。
3. **缓解手段**：SLO 攸关分区可被**显式 pin 为备区域的 always-active**，从而完全避免冷启动。
4. **与 [[Hot Backup]] 的对称性**：热备解决区域内节点级恢复速度，always-active pin 解决区域级冷启动，二者是同一种「用资源换恢复时间」的思路。

## 代表工作
- [[K2Plus]]：区域故障实验中，副本提升在 110 s 立即完成，cold start 额外约 27 s。

## 相关概念
- [[Region Failover]]
- [[Hot Backup]]
- [[Regional Manager]]
- [[Replication Middleware]]
