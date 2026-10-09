---
type: concept
aliases: [recall amplification factor, 放大因子, sigma]
---

# Recall Amplification Factor

## 定义

在向量索引中先检索出的候选集大小相对于最终结果数 $k$ 的倍率，记为 $\sigma$，用于在过滤或后处理之前扩大候选池以保证 recall。

## 数学形式

$$
\text{候选集大小} = \sigma \times k
$$

## 核心要点

1. **动机**：混合查询中若先取 $k$ 个向量再过滤，可能因大量候选被谓词剔除而拿不满 $k$ 个结果，因此需要放大候选池。
2. **静态 $\sigma$ 的缺陷**：在计划期固定 $\sigma$ 无法适应运行时数据分布——过滤强时不足、过滤弱时做无用功。
3. **改进**：[[Nova]] 的 [[Iterative Query|迭代查询]]机制不再固定 $\sigma$，而是增量扩张搜索前沿直到获得 $k$ 个通过过滤的结果；这属于 Type 3 后过滤流水线的精化。
4. **用户可配**：Nova 的 `scan` 也允许用户通过可配的放大因子进一步提升 recall。

## 代表工作

- [[Nova]]: 同时用于纯向量检索的代价估计（预期检索量 $k \times \sigma$）与 [[ANN-driven Hash Join]] 的代价模型。

## 相关概念

- [[Iterative Query]]
- [[Hybrid Search]]
- [[ANNS]]
