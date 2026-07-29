---
type: concept
aliases: [分离段索引布局, Detached Segment Index Layout]
---

# Detached-Segment Index Layout

## 定义

一种面向长序列混合访问的数据布局：将每个实体的记录按时间切成独立数据段，索引只保存各段的地址、时间范围与记录数，从而同时支持高效点读和范围扫描。

## 核心结构

1. 以实体 ID（如 user ID）为 key 聚合该实体的全部有序记录。
2. 将大 value 按时间范围切为多个 segment，顺序存入 [[ValueLog]]。
3. 在 [[SSTable]] 中仅保存 segment 元数据，使索引紧凑、易缓存。
4. 查询先在索引中过滤目标 segment，再读取数据，避免全量扫描与跨分区 join。

## 关键权衡

- segment 越小，时间过滤越精确，但索引和随机 I/O 越多。
- segment 越大，顺序读效率越高，但读取短窗口时放大更明显。
- compaction 可能打散同一实体的数据，需要 [[Locality-Triggered Garbage Collection]] 恢复局部性。

## 代表工作

- [[TokaDB]]：用于统一支持用户行为序列的 serving point read 与 training range scan。

## 相关概念

- [[KV Separation]]
- [[ValueLog]]
- [[SSTable]]
- [[User Behavior Sequence]]
