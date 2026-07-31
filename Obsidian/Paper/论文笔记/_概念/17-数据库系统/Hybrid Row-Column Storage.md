---
type: concept
aliases: [行列混合存储, Hybrid Row-Column Storage]
---

# Hybrid Row-Column Storage

## 定义

同时维护行式和列式物理组织，并根据查询读取列数选择访问路径的存储设计。

## 核心要点

1. 该概念用于描述论文中的关键数据模型、算法或系统组件。
2. 具体实现与性能结论应以关联论文的实验条件为准。

## 代表工作

- [[LindormTSDB]] / [[LindormUWC]]：相关设计或评测语境。

## 相关概念

- [[LindormUWC]]
- [[Parquet]]
- [[SSTable]]
