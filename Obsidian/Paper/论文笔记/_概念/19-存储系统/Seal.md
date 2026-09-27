---
type: concept
aliases: [日志封闭, Sealing Primitive, I/O Fencing]
---
# Seal

## 定义
Seal（sealing 原语）是存储层提供的一种操作：**阻止对某个日志段的后续写入**，使其固化。它是 [[K2Plus]] 在节点故障切换时建立唯一恢复边界、防止 [[Split-Brain]] 的关键机制。

## 核心要点
1. **为什么必须有**：[[K2Plus]] 的恢复依赖新旧节点**共享同一份 LSM 状态**（数据在共享存储池里）以加速恢复。如果不封禁旧节点的写路径，两个节点会同时修改同一棵 LSM 树。
2. **seal 序列**：新数据节点接管分区前，按 **Root Plog → Index Plog → WAL Plog** 的顺序对底层 Plog 发出 Seal。封禁旧 WAL 后不再有追加生效，旧写者无法修改 LSM 树，于是 seal **定义了一个稳定的恢复边界**，实现轻量级的「每个 Plog 单写者」。
3. **只解决写分叉**：读可以不经持久层直接从内存返回，因此与 [[Lease]] 检查配合，才能阻止故障节点返回陈旧数据。注意 lease 检查**只能在读之后做**——写路径无法在持久化请求之后补校验。
4. **依赖假设**：区域内存储层完全受管且只能通过暴露的 API 访问。对不支持原生 Seal 的日志系统，可在不透明存储 API 之上实现通用 fencing 层，但更复杂且低效（需跟踪所有权与在途请求）。
5. **同类原语**：Azure Append Blob、[[Delos]]、[[Apache HBase]] 等系统提供类似的 fencing 能力。

## 代表工作
- [[K2Plus]]：把 DFV 的 I/O fencing API 与事务协议协同设计，是「持久化与事务协议协同设计」这一贡献的组成部分。

## 相关概念
- [[Plog]]
- [[DFV]]
- [[Split-Brain]]
- [[Lease]]
- [[Partition Migration]]
