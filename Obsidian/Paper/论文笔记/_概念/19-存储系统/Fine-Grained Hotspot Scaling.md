---
type: concept
aliases: [细粒度热点扩展, Hot-Key Selective Replication]
---

# Fine-Grained Hotspot Scaling

## 定义

只在 follower 上回放和重建热点 key 页面，并将读请求路由到已缓存相应热页副本的选择性复制机制。

## 核心要点

1. 扩展粒度是 key/[[Adjacency List|邻接表]]，而非整个节点或分区。
2. follower lazy-apply 目标热点的 WAL，不重建冷 key 页面。
3. 热点感知路由缩小每个副本的 working set，提高 buffer pool 命中率。
4. 热度进一步倾斜时，可为单个 dominant hot key 增加专用 follower。
5. [[ByteGraph-Dione]] 实验中页面重建减少 98.32%，cache hit 达 90%–93%。

## 代表工作

- [[ByteGraph-Dione]]：面向持续且动态的图邻接表热点提出该机制。

## 相关概念

- [[Adjacency List]]
- [[ByteGraph]]
- [[Disaggregated Storage-Compute Architecture]]
