---
type: concept
aliases: [增量视图维护, IVM]
---

# Incremental View Maintenance

## 定义

根据基础数据变化量更新物化查询结果，而非每次从头执行完整查询。

## 核心要点

1. 维护成本通常与 delta 和受影响状态相关。
2. 可撤回聚合可直接加减，非可撤回聚合需局部重算。
3. join 需要传播 lineage 并处理重复与外连接修正。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Incremental Processing Mode]]
- [[MVCC]]
