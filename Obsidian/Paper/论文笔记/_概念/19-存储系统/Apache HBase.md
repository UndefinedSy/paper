---
type: concept
aliases: [HBase]
---
# Apache HBase

## 定义
Apache HBase 是构建在 HDFS 之上的分布式、可扩展的**列式键值存储**，其数据模型脱胎于 [[Bigtable]]，依赖 ZooKeeper 做元数据与协调、依赖 HDFS 做持久化。

## 核心要点
1. **架构分层**：HMaster 管元数据与负载均衡，RegionServer 承载 region（数据分片），ZooKeeper 负责协调与故障探测，HDFS 提供多副本持久化。
2. **一致性模型**：单个 region 内的行级操作是原子的；跨 region 的强一致事务需要额外组件（如 HBase 自身不提供跨行事务）。
3. **[[K2Plus]] 的引用方式**：论文把 HBase 与 [[Delos]] 一起列为「提供类似 fencing 能力」的系统例子，用于说明 K2+ 对存储层的 sealing 要求不是 DFV 专有。
4. **与 K2+ 的架构对照**：HBase 的 region 与 RegionServer 有绑定关系，故障恢复需重放 HLog 并重新分配 region，是典型的**计算与存储未解耦**架构；K2+ 的物理分区则在共享存储池之上，恢复只需切换元数据与新写路径。

## 代表工作
- [[K2Plus]]：作为具备 fencing 类原语的同类系统被引用。
- [[Bigtable]]：HBase 的直接思想来源。

## 相关概念
- [[Bigtable]]
- [[HDFS]]
- [[ZooKeeper]]
- [[Seal]]
- [[Tablet]]
