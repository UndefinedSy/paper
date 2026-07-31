---
type: concept
aliases: [Join 下推, 分布式 Join 优化]
---
# Join Pushdown
## 定义
把 join 运算下推到数据所在节点本地执行、只在上层汇总结果的优化，避免跨节点的数据重分布（shuffle）。
## 核心要点
1. 最有效的场景是**共置表之间的 inner join**（[[Data Co-location]]）：各分片并行做局部 join，上层只需 append。
2. 分片表 × 参考表同样可下推，因为参考表在每个 shard 上都有完整副本。
3. ⚠️ **仅适用于特定 join 类型**：笛卡尔积、inner join、参考表为 null-padded 侧的 outer join；**不适用**于分片表为 null-padded 侧的 outer join 与 anti-join。
4. [[Aurora Limitless Database]] 依赖 PostgreSQL 的 partition-aware join 算法，但因原生 cost model 不知数据在远端而**修改了代价模型**。
5. 能力缺口：standard 表与其它类型表的 join **目前只能在 router 上执行**（完全不下推）。
## 代表工作
- [[AuroraLimitless]]：复用 partition-wise join 并修改代价模型。
## 相关概念
- [[Data Co-location]]
- [[Predicate Pushdown]]
- [[Foreign Data Wrapper]]
- [[Sharding]]
