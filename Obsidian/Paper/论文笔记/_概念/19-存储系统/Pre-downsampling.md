---
type: concept
aliases: [预降采样, Pre-downsampling]
---

# Pre-downsampling

## 定义

在写入、flush 或 compaction 阶段预先计算固定时间粒度的聚合结果，以减少查询时扫描的原始点。

## 核心要点

1. 该概念用于描述论文中的关键数据模型、算法或系统组件。
2. 具体实现与性能结论应以关联论文的实验条件为准。

## 代表工作

- [[LindormTSDB]] / [[LindormUWC]]：相关设计或评测语境。

## 相关概念

- [[LindormTSDB]]
- [[Time Series]]
