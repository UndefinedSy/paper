---
type: concept
aliases: [Persistent Log, 持久日志]
---
# Plog

## 定义
Plog（persistent log）是 [[DFV]] 提供的存储抽象：一段**连续的、append-only 的存储区段**，通过 Plog client 访问，是 [[K2Plus]] 持久化层的入口。

## 核心要点
1. **层级结构**：每个 [[Physical Partition|物理分区]] 有一个 **Root Plog** 作为持久化层入口；Root Plog 关联若干 **Index Plog**，记录 [[WAL]] Plog 与 LSM 各层的偏移；其上还有 Write-ahead Plog、Level 0/1/2 Plog（Figure 7）。
2. **单写者语义**：靠 [[Seal]] 原语保证每个 Plog 同一时刻只有一个写者，是防 [[Split-Brain]] 的基石。
3. **恢复边界**：节点故障迁移时，新节点**依次 Seal Root Plog → Index Plog → WAL Plog**，阻止旧节点继续写入，从而定义一个稳定的恢复边界；之后新节点切换 Root Plog 并创建新 Index Plog，重建并索引既有 LSM 树。
4. **可移植要求**：[[K2Plus]] 对区域存储的全部要求就是「append-only 持久日志抽象 + sealing 原语」两条。

## 代表工作
- [[K2Plus]]：数据节点经 Plog client 把 WAL 与 LSM 各层数据写入 DFV。

## 相关概念
- [[DFV]]
- [[Seal]]
- [[WAL]]
- [[Split-Brain]]
- [[LSM-tree]]
