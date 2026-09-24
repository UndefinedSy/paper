---
type: concept
aliases: [自适应压实]
---

# Adaptive Compaction

## 定义

根据增量段积压、写入压力或查询局部性动态调整压实频率、批量和优先级的存储维护机制。

## 核心要点

1. 在写放大与碎片化之间动态权衡。
2. 控制器应平滑改变强度，避免压实振荡和摄入吞吐骤降。
3. ByteHouse 用有界线性控制器把 delta segment 数映射为压实强度。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Shared-Storage Architecture]]
- [[MVCC]]
