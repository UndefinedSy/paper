---
type: concept
aliases: [ANN-driven Hash Join]
---

# ANN-driven Hash Join

## 定义

跨表混合检索中的一种执行策略：优先执行向量检索，用 ANN 扫描结果构造哈希表，再由过滤后的元数据表逐行探测。

## 数学形式

$$
Cost_{HJ} \propto \min\left(k \cdot \sigma \cdot \frac{N}{N_f},\ S_{max}\right)
$$

## 核心要点

1. 性能由**初始 ANN 扫描的规模**主导。
2. $k$ 为 top-$k$ 限制，$\sigma$ 为 [[Recall Amplification Factor|放大因子]]，$N$ 为 embedding 表总行数，$N_f$ 为满足元数据过滤的估算行数。
3. $N/N_f$ 即**逆过滤选择率**；$S_{max}$ 是系统定义的可扫描点数上限。
4. 适合元数据过滤**选择性较弱**的场景——一次大规模 ANN 扫描比多次小查找更经济。

## 代表工作

- [[Nova]]: 作为跨表混合检索的两个候选策略之一，与 [[Metadata-driven Join]] 由 [[Cost-Based Adaptive Join]] 动态选择。

## 相关概念

- [[Cost-Based Adaptive Join]]
- [[Metadata-driven Join]]
- [[ANNS]]
