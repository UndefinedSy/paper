---
type: concept
aliases: [Virtual Finish Time, VFT, 虚拟完成时间]
---

# Virtual Finish Time (VFT)

## 定义

WFQ（加权公平队列）中的虚拟时间戳，用于表示一个请求在"理想 GPS 调度器"中应当完成的虚拟时间点，是实现公平调度的核心度量。

## 数学形式

$$\text{VFT}(Q_i) = \text{preVFT}_{T_i} + \text{wReqCost}(Q_i)$$

其中加权请求代价：

$$\text{wReqCost}(Q_i) = \frac{\text{Cost}(Q_i)}{Q_i / \sum Q_p}$$

## 核心要点

1. VFT 越小的请求越先被处理（最小堆）
2. 同一租户的 VFT 单调递增（累积），防止短期内同一租户的多个请求都拥有最小 VFT
3. 请求代价按分区配额比例加权，配额占比小的租户单次请求"代价"更高，意味着它们的队列推进更慢，更容易被选中

## 代表工作

- [[ABase]]: Dual-Layer WFQ 的调度核心
- [[2DFQ]]: 理论基础

## 相关概念

- [[WFQ|Weighted Fair Queueing]]
- [[2DFQ]]
