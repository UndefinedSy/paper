---
type: concept
aliases: [Locality Group, 本地性组]
---

# Locality Group

## 定义

Bigtable 中 column family 的进一步分组，每个 locality group 构成存储的基本单位，在 compaction 时为每个 locality group 生成独立的 [[SSTable]]。

## 核心要点

1. Column family 进一步分组为 locality group
2. 每个 locality group 在 compaction 时生成独立 [[SSTable]]
3. 构成存储的基本单位
4. 复制日志是系统 locality group（对用户隐藏）
5. External compaction 允许 locality group 级别的并行处理

## 代表工作

- [[Bigtable]]: 列存储分组和压缩单位

## 相关概念

- [[Bigtable]]
- [[SSTable]]
- [[Tablet]]
