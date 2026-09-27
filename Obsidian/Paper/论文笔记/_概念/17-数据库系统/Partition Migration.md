---
type: concept
aliases: [分区迁移, Partition Rescheduling]
---
# Partition Migration

## 定义
Partition Migration 是把数据分区从**故障节点或过载节点**搬迁到同区域内**有容量的节点**的过程。在存储解耦架构中，因为数据本身在共享存储池里，迁移主要是元数据与新写路径的切换，而非搬运数据。

## 核心要点
1. **[[K2Plus]] 的迁移流程**（Figure 11）：
   - ① [[Regional Manager]] 更新分区放置信息；
   - ② 在欠载节点上构建新的数据分区；
   - ③ 从共享 [[DFV]] 恢复旧 [[WAL]] 与最新 SSTable；
   - ④ 新节点 flush 重建的 SSTable，并以新 WAL 开始服务。
2. **新旧节点共享 LSM 状态**：这是恢复快的原因，但也是必须对旧 Plog 执行 [[Seal]] 的原因——否则两个节点会同时写同一棵 LSM 树。
3. **不恢复在途事务**：失效节点上的 [[Read Tracker]] 与 [[Write Stream]] 不持久化，在途事务一律视为已中止（强行恢复会削弱一致性）。
4. **同区域优先**：区域故障罕见 + 维持数据局部性，因此 K2+ 总是在同区域恢复，并**常备若干空闲热备节点**以加速。
5. **拆分复用同一路径**：[[Partition Split]] 后的迁移可跳过 WAL 填充/flush，只搬索引与元数据，数秒内完成。

## 代表工作
- [[K2Plus]]：节点故障场景下检测后约 3 s 恢复到故障前吞吐；弹性场景下拆分+迁移 3 s 内完成。

## 相关概念
- [[Partition Split]]
- [[Physical Partition]]
- [[Regional Manager]]
- [[Seal]]
- [[Hot Backup]]
