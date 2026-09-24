---
type: concept
aliases: [集群级分布式缓存]
---

# CrossCache

## 定义

ByteHouse 的存储侧集群共享 SSD 缓存，通过协调节点维护命名空间、缓存节点承载数据块，缓解远端对象存储延迟。

## 核心要点

1. 文件默认切为 12 MB block，再切为 4 MB chunk。
2. 跨计算节点共享热点数据，减少调度导致的缓存局部性损失。
3. 并行 flush 与 concat 提高 HDFS/对象存储写吞吐。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Shared-Storage Architecture]]
- [[NexusFS]]
