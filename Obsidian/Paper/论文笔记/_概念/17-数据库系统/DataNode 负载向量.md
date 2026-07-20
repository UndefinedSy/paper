---
type: concept
aliases: [DataNode 负载向量, Load Vector, 负载向量]
---

# DataNode 负载向量

## 定义

ABase 资源重调度中用于表征 DataNode 负载分布的时序向量，将过去 7 天的逐小时平均负载按每天小时维度聚合取峰值，消除绝对日期差异并保留日内周期性规律。

## 数学形式

副本负载向量：$RE^{ld} = (RE^{ld}_1, \cdots, RE^{ld}_{24})$

DataNode 负载（所有副本叠加后取全天峰值）：

$$DN^{ld} = \max_i \left( \sum_{RE \in DN} RE^{ld}_i \right) \quad \text{for } i \in \{1,2,...,24\}$$

## 核心要点

1. 使用过去 7 天数据避免短暂波动影响，取每小时最大值体现峰值压力
2. 按 hour-of-day 维度折叠，消除"某天特殊流量"影响，反映稳定日内周期模式
3. RU 负载向量融合读 RU、写 RU 和缓存命中率的加权

## 代表工作

- [[ABase]]: Intra/Inter-Pool 重调度的核心负载度量

## 相关概念

- [[迁移收益函数]]
- [[Resource Pool]]
