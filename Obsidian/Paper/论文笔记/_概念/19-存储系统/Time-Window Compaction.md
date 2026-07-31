---
type: concept
aliases: [时间窗口合并, Time-Window Compaction]
---

# Time-Window Compaction

## 定义

只在相同时间窗口内合并不可变文件的 compaction 策略，适合时间追加型数据。

## 核心要点

1. 该概念用于描述论文中的关键数据模型、算法或系统组件。
2. 具体实现与性能结论应以关联论文的实验条件为准。

## 代表工作

- [[LindormTSDB]] / [[LindormUWC]]：相关设计或评测语境。

## 相关概念

- [[LindormUWC]]
- [[LSM-tree]]
- [[Hot-Cold Tiering]]
