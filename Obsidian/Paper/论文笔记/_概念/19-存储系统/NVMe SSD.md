---
type: concept
aliases: [固态硬盘, Solid State Drive]
---

# NVMe SSD

## 定义

基于 NVMe（Non-Volatile Memory Express）协议的固态存储设备，相比传统 SATA/SAS 接口 SSD 提供更高带宽与更低访问延迟（典型 KB 级随机读约数百微秒），常用于低延迟存储系统的物理介质。

## 核心要点

1. KB 级随机读延迟（约 700μs，[[SOS]] 论文数据）与数据中心内网络 RTT（约 1ms）量级相近，使"磁盘驻留 + 网络访问"的延迟开销可以逼近"内存驻留 + 网络访问"，这是 SOS 等系统能在 SSD 成本下达到接近内存存储延迟的物理基础。
2. 相比 DRAM，NVMe SSD 每 GB 成本显著更低（约 1/10 到 1/30），是 PB 级存储系统选用的主要动机。
3. 对延迟敏感的 KV 存储系统，SSD 驻留架构的设计目标通常是把"每次操作的物理 I/O 次数"降到最少（理想为 1 次），从而最大化利用 SSD 的低延迟特性。

## 代表工作

- [[SOS]]：核心面向的存储介质，评估硬件为 Samsung PM9A3 NVMe SSD（3.84TB，4KB native LBA）。

## 相关概念

- [[Direct IO]]
- [[Hash Index]]
