---
type: concept
aliases: [邻接表, Adjacency-List Storage]
---

# Adjacency List

## 定义

图存储中按顶点组织其相邻边/邻居集合的数据结构，适合从给定顶点开始做一跳或多跳遍历。

## 核心要点

1. 常把 source vertex 作为分区或索引入口，边按 destination/edge key 排序。
2. 小范围点查适合行式布局，大范围属性扫描可能更适合列式布局。
3. 图的度数和访问常呈幂律分布，因此不同邻接表在容量、查询类型和热度上差异很大。
4. [[ByteGraph-Dione]] 以单条邻接表作为格式转换和热点复制粒度。

## 代表工作

- [[ByteGraph-Dione]]：邻接表级自适应双格式与细粒度扩缩容。
- [[TAO]]：面向社交图的关联列表存储。

## 相关概念

- [[BW-Tree]]
- [[Adaptive Dual-Format Graph Storage]]
- [[Fine-Grained Hotspot Scaling]]
