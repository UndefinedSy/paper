---
type: concept
aliases: [DFV, Data Functions Virtualization, 华为云存储池]
---
# DFV

## 定义
DFV（Data Functions Virtualization）是**华为云统一数据存储平台**，于每个地理区域内提供弹性、高可靠、高可用的存储池，是 [[K2Plus]] 实现区域内存储解耦的物理基座。

## 核心要点
1. **覆盖面**：支撑华为云上从对象/块/文件存储到数据库与大数据分析的全部数据类服务。
2. **部署形态**：存储节点横跨 **3 个 [[Availability Zone]]**；单 AZ 内靠多副本 + 后台 [[Erasure Coding|纠删码]]，跨 AZ 靠**同步写三副本**。
3. **持久性–可用性权衡**：三 AZ 全部落盘后才 ACK，比 quorum 方案提供更强的区域持久性；代价是存储节点故障修复期间该组写入会短暂阻塞（活副本仍可服务读）。生产指标：**11 个 9 持久性、5 个 9 可用性**。
4. **核心抽象**：[[Plog]]（persistent log，连续 append-only 存储区段）+ Plog client + [[Seal]] 原语。
5. **对 [[K2Plus]] 的两个支撑点**：提供区域内可靠落盘（WAL 与 LSM 各层数据），以及提供 Seal 以建立唯一的恢复边界防脑裂。
6. **可移植性**：K2+ 只要求区域存储服务提供 (1) append-only 持久日志抽象、(2) sealing 原语；两者的通用实现可参考 Azure Append Blob、[[Delos]] 等。消融显示换用 Local File 后端时 K2+ 功能仍然完整，DFV 主要贡献延迟与运维健壮性。

## 代表工作
- [[K2Plus]]：以 DFV 为区域内存储池，实测 DFV 相较本地文件吞吐开销 < 2%，追加延迟 300 μs（本地文件 < 100 μs）。

## 相关概念
- [[Plog]]
- [[Seal]]
- [[Availability Zone]]
- [[Erasure Coding]]
- [[K2Plus]]
