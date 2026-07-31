---
type: concept
aliases: [FDW, 外部数据包装器]
---
# Foreign Data Wrapper
## 定义
PostgreSQL 的扩展接口：让 planner 与 executor 能够像访问本地表一样访问外部数据源中的数据。
## 核心要点
1. 实现一个 FDW 需提供扫描、代价估算、下推判定等回调，planner 据此把外部表纳入统一的优化框架。
2. [[Aurora Limitless Database]] 实现了**自定义 FDW** 供 router 访问 shard 数据：分片表的每个分区、参考表、standard 表都表示为 foreign table。
3. **设计红利**：借壳 PostgreSQL 已理解的抽象（分区表 + 外部表），从而直接复用其 planner 做部分聚合、排序、[[Predicate Pushdown]] 与 [[Join Pushdown]]。
4. 配合 **async foreign scan** 即可实现各 shard 的并行执行。
5. ⚠️ 代价模型不知道数据实际在远端 shard 上，因此 Limitless **修改了 PostgreSQL 的 cost model**。
## 代表工作
- [[AuroraLimitless]]：以自定义 FDW 表达全部分片数据。
- [[PostgreSQL]]：FDW 接口的提供者。
## 相关概念
- [[PostgreSQL]]
- [[Predicate Pushdown]]
- [[Join Pushdown]]
- [[Partition Pruning]]
