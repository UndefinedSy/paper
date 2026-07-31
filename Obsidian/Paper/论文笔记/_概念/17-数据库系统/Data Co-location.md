---
type: concept
aliases: [共置, Colocation, 数据共置]
---
# Data Co-location
## 定义
让共享同一 shard key 的多张表中「同一 key 值的数据」落在同一节点上的放置策略，目的是把跨表 join 变成节点本地操作。
## 核心要点
1. 没有共置时，两张分片表的 join 需要跨节点重分布数据（shuffle），代价高昂。
2. 共置后，inner join 可以在每个分片上并行局部完成，结果只需在上层 append。
3. [[Aurora Limitless Database]] 通过 `limitless_create_table_collocate_with` 声明，并保证共置表的对应 [[Table Slice]] 在 shard split 时**一起迁移**，从而在扩容后仍保留 join 优化。
4. 代价：共置约束限制了各表独立选择分片键的自由度。
## 代表工作
- [[AuroraLimitless]]：session 参数声明共置 + slice 联动迁移。
## 相关概念
- [[Sharding]]
- [[Table Slice]]
- [[Join Pushdown]]
- [[Partition Pruning]]
