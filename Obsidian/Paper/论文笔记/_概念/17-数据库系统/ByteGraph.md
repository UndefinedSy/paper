---
type: concept
aliases: [ByteDance Graph Database, 字节跳动图数据库]
---

# ByteGraph

## 定义

ByteDance 的分布式图数据库，采用执行—存储解耦架构，在 append-only 共享存储上以 [[BW-Tree]] forest 组织 vertex-centric [[Adjacency List|邻接表]]。

## 核心要点

1. 每个 vertex 映射到一棵 BW-Tree，出边按 edge key 排序。
2. 通过 leader/follower、WAL 与共享存储提供一致的实时图分析。
3. 上一代固定使用行式 LeafPage，偏向 OLTP。
4. [[ByteGraph-Dione]] 在其上加入轻量事务、自适应双格式和热点细粒度扩展。

## 代表工作

- [[ByteGraph-Dione]]：面向生产级混合 OLTP/OLAP 的新一代引擎。

## 相关概念

- [[BW-Tree]]
- [[Adjacency List]]
- [[Disaggregated Storage-Compute Architecture]]
- [[HTAP]]
