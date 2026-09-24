---
type: concept
aliases: [增量处理模式, IPM]
---

# Incremental Processing Mode

## 定义

ByteHouse 跨刷新周期保留算子状态、只处理新增和撤回 delta 的持续查询执行模式。

## 核心要点

1. tuple_key 与 update_seq 提供行级 lineage。
2. 更新统一表示为旧版本 delete 加新版本 insert。
3. 聚合和 join 通过可组合撤回维护结果。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Incremental View Maintenance]]
- [[MVCC]]
