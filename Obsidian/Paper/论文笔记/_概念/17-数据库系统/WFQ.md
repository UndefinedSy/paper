---
type: concept
aliases: [Weighted Fair Queueing, WFQ, 加权公平队列]
---

# WFQ (Weighted Fair Queueing)

## 定义

加权公平队列（WFQ）是一种调度算法，通过为每个流（或租户）分配虚拟完成时间（VFT），按权重比例公平分配处理资源，防止大流量流垄断共享资源。

## 数学形式

ABase 中的 VFT 计算：

$$\text{wReqCost}(Q_i) = \frac{\text{Cost}(Q_i)}{Q_i / \sum Q_p}$$

$$\text{VFT}(Q_i) = \text{preVFT}_{T_i} + \text{wReqCost}(Q_i)$$

队列为最小堆，优先处理 VFT 最小的请求。

## 核心要点

1. 按配额比例加权请求代价，配额占比越小的租户单次请求权重越高
2. VFT 在同租户请求间累积，防止高配额租户始终优先
3. ABase 的 Dual-Layer WFQ：CPU-WFQ（命中缓存直返）+ IO-WFQ（未命中走磁盘）
4. 参考 2DFQ 将请求按类型（读/写）× 大小（大/小）分为 4 组独立队列，避免大小请求互相干扰

## 代表工作

- [[ABase]]: Dual-Layer WFQ，区分 CPU 和 I/O 两层调度，结合缓存感知
- [[2DFQ]]: 二维公平队列，ABase WFQ 的理论基础

## 相关概念

- [[Virtual Finish Time (VFT)]]
- [[请求单元 (RU)]]
- [[2DFQ]]
