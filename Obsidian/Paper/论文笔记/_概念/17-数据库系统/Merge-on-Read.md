---
type: concept
aliases: [读时合并, Merge-on-Read]
---

# Merge-on-Read

## 定义

把多个针对同一逻辑记录的增量在读取或 compaction 时合并，而不是每次写入时重写完整记录。

## 核心要点

1. 该概念用于描述论文中的关键数据模型、算法或系统组件。
2. 具体实现与性能结论应以关联论文的实验条件为准。

## 代表工作

- [[LindormTSDB]] / [[LindormUWC]]：相关设计或评测语境。

## 相关概念

- [[LindormUWC]]
- [[LSM-tree]]
