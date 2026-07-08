---
type: concept
aliases: [最终一致性, Eventual Consistency Model]
---

# Eventual Consistency

## 定义

一种一致性模型，保证在给定 replica 的复制水位线中，给定 source 在给定 row 的复制时间点不会倒退，且不同 source 在同一 row 上的 mutation 全局排序，冲突通过 last-write-wins 解决。

## 核心要点

1. 保证复制时间点单调递增，不回退
2. 不同 source 的 mutation 通过 sequencer（replication timepoint + source ID）全局排序
3. 冲突解决采用 last-write-wins 策略
4. ACID 保证仅限于单行，跨行/跨 replica 无保证
5. 优势：低成本、高吞吐、低延迟、写仅需单个 replica
6. 常见使用模式：流量隔离、多主写入、故障切换

## 代表工作

- [[Bigtable]]: 复制一致性模型
- [[Cassandra]]: 去中心化最终一致性
- [[DynamoDB]]: 基于 Dynamo 的最终一致性

## 相关概念

- [[Bigtable]]
- [[CRDT]]
- [[Spanner]]
