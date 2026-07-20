---
type: concept
aliases: [2DFQ, Two-Dimensional Fair Queueing, 二维公平队列]
---

# 2DFQ (Two-Dimensional Fair Queueing)

## 定义

2DFQ 是 SIGCOMM 2016 提出的多租户云服务调度算法，在标准 WFQ 基础上引入第二维度（如请求大小），防止大请求和小请求之间的相互干扰，保证跨租户的公平调度。

## 核心要点

1. 标准 WFQ 对"大请求"和"小请求"使用统一队列，大请求会阻塞小请求，导致小请求延迟不可控
2. 2DFQ 将请求按大小（或其他维度）分组，独立调度，确保各类请求延迟可预期
3. ABase 参考此思想，将请求分为 Small Read/Large Read/Small Write/Large Write 四类独立 Dual-Layer WFQ

## 代表工作

- Mace et al., SIGCOMM 2016: "2DFQ: Two-Dimensional Fair Queuing for Multi-Tenant Cloud Services"
- [[ABase]]: 基于 2DFQ 思想设计 4 组独立双层 WFQ

## 相关概念

- [[WFQ|Weighted Fair Queueing]]
- [[Virtual Finish Time (VFT)]]
