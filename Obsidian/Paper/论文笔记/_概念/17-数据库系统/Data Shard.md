---
type: concept
aliases: [数据分片, Data Shard]
---
# Data Shard

## 定义
Data Shard 是 [[K2Plus]] 中**与 CPU 核绑定**的资源分区单位：[[Seastar]] 按核分配资源（每核独立内存）以避免跨核争用，K2+ 沿用该模型把每个核绑定为一个 data shard。

## 核心要点
1. **核即分片**：每个 data shard 拥有独立内存、独占一个 CPU 核，因此 shard 间无需锁保护共享状态。
2. **shard 与分区的解耦**：核绑定带来问题——动态负载下 resharding 代价高昂，因为核间内存空间不同。K2+ 的解法是在 shard 与 [[Physical Partition|物理分区]]之间加入路由层，使物理分区成为最小管理单元，一个 shard 可容纳多个物理分区。
3. **容量约束**：物理分区分 large/medium/small 三档，分别消耗 100%/50%/25% CPU，因此**每个 shard 最多 4 个物理分区**，以避免存储碎片与路由复杂度失控。
4. **细粒度均衡的收益**：因为一个核可承载多个分区，[[Partition Migration]] 可以只搬走热点分区的**一部分**，把负载摊平到各核，而非整 shard 迁移。

## 代表工作
- [[K2Plus]]：Data Node A 内 Core 0–3 各对应 Shard#0–#3，P#0–P#7 作为物理分区映射其上；均衡后各数据节点 CPU 利用率超 80%。

## 相关概念
- [[Physical Partition]]
- [[Seastar]]
- [[Partition Migration]]
- [[Load Balancing]]
