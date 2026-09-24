---
type: concept
aliases: [魔集重写]
---

# Magic Set Rewriting

## 定义

把查询约束向递归或上游子计划传播、提前限制中间关系的查询重写技术。

## 核心要点

1. 通过复制选择性子计划实现早过滤。
2. 可显著减少中间结果。
3. 需要权衡重复计算与裁剪收益。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Cascades Optimizer]]
- [[Runtime Filter]]
