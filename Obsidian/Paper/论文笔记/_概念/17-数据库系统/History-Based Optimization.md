---
type: concept
aliases: [历史驱动优化, HBO]
---

# History-Based Optimization

## 定义

复用过去查询执行的真实统计信息来修正未来计划代价估计的优化方法。

## 核心要点

1. canonical plan fragment 通过 hash 匹配历史记录。
2. 可复用 predicate selectivity 与 join cardinality。
3. 对未见查询模式的泛化较弱，常与学习模型结合。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Cascades Optimizer]]
- [[Predicate Pushdown Selection]]
- [[Join Side Selection]]
