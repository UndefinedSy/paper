---
type: concept
aliases: [物理分区]
---
# Physical Partition

## 定义
Physical Partition 是 [[K2Plus]] 中**最小的数据管理单元**：由某张表的一组行组成，从存储视角对应**一棵独立的 [[LSM-tree]]**。

## 核心要点
1. **与 shard 的关系**：路由层把物理分区映射到 [[Data Shard]]，一个 shard 最多承载 4 个物理分区。
2. **三档规格**：large / medium / small 分别设计为消耗 **100% / 50% / 25% CPU 资源**。这既限制了分区数量（避免 LSM 碎片与路由复杂度），又允许细粒度的负载搬运。
3. **与逻辑分区（global range / regional range）的区别**：逻辑分区决定**路由**，物理分区决定**存储与计算承载**。[[K2Plus]] 的关键设计是**物理分区信息只在所属区域内可见**，不跨区域复制元数据。
4. **N-to-M 复制关系**：因为各区域独立切分物理分区，主区域与副本区域的物理分区对应关系是 N-to-M，而非严格一对一（Figure 8 中 R1 的 4 个分区对应 R2 的 1 个分区）。
5. **元数据**：`PartitionID | ShardID | TableID | Key_min | Key_max | Size | PLogIDs`。

## 代表工作
- [[K2Plus]]：物理分区作为最小管理单元，使 region-local 的 split/merge/迁移成为纯本地操作。

## 相关概念
- [[Data Shard]]
- [[Partition Split]]
- [[Partition Migration]]
- [[Replication Middleware]]
- [[LSM-tree]]
