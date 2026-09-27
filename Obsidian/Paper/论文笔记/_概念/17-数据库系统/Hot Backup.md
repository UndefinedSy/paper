---
type: concept
aliases: [热备节点, Hot Standby]
---
# Hot Backup

## 定义
Hot Backup（热备）是系统中**预先保持空闲、可立即接管工作**的节点。[[K2Plus]] 始终维护若干活跃的空闲数据节点，用于加速故障恢复与负载均衡。

## 核心要点
1. **为什么需要**：数据节点故障后需要把分区分区迁移到新节点（见 [[Partition Migration]]）；若没有现成空闲节点，还需要先申请/启动实例，恢复时间会被拉长。
2. **与 [[Cold Start]] 的区别**：热备节点已运行、已接入管理面（有心跳、在集群视图内），接管分区只需切换元数据与存储对象；cold start 则要从零重建内存组件（memtable、索引）并注册。
3. **代价**：常备空闲资源意味着一定的资源冗余开销，是**恢复时间 vs. 资源成本**的权衡。
4. [[K2Plus]] 的做法：既维护热备节点以加速区域内恢复，又在区域级对 SLO 攸关分区采取「always-active in backup region」的 pin 策略，把关键分区的冷启动成本降到零。

## 代表工作
- [[K2Plus]]：为快速恢复与负载均衡常备活跃空闲数据节点。

## 相关概念
- [[Partition Migration]]
- [[Cold Start]]
- [[Load Balancing]]
- [[Region Failover]]
