---
type: concept
aliases: [Cascades 查询优化器]
---

# Cascades Optimizer

## 定义

用 memo、等价变换规则和代价模型统一搜索逻辑与物理查询计划的优化框架。

## 核心要点

1. memo 避免重复枚举等价表达式。
2. required/derived physical properties 约束 partition、sort 与 group。
3. ByteHouse 在其上加入 bushy join 枚举、Magic Set 和 CTE 决策。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Magic Set Rewriting]]
- [[History-Based Optimization]]
