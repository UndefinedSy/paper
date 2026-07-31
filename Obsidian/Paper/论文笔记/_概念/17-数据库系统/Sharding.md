---
type: concept
aliases: [分片, 水平分区, Horizontal Partitioning]
---
# Sharding
## 定义
把一张表的数据按某个 shard key 水平切分到多个独立节点上，从而突破单机的写吞吐、存储容量与连接数上限。
## 核心要点
1. 最常见的划分方式是对 shard key 做哈希（均衡但丧失范围局部性）或按范围切分（保留范围查询但易产生热点）。
2. shard key 的选择是**单向门**：改变它需要重新分片并迁移全量数据，因此是迁移中最关键的设计决策。
3. 应用层分片要求业务自己处理路由、跨分片事务与 schema 演进，运维负担极重；透明分片系统的价值就在于把这些收进数据库。
4. 共置（[[Data Co-location]]）是让 join 保持本地性的关键手段。
5. [[Aurora Limitless Database]] 在 shard 之下再引入 [[Table Slice]] 作为更细的迁移单元。
## 代表工作
- [[AuroraLimitless]]：透明分片 + 三种表类型 + slice 级迁移。
- [[Citus]] / [[Vitess]] / [[Apache ShardingSphere]]：其它分片方案。
## 相关概念
- [[Data Co-location]]
- [[Table Slice]]
- [[Two-Phase Commit]]
- [[Partition Pruning]]
