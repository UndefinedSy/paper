---
type: concept
aliases: [冷热分层, SSD-HDD Hot-Cold Tiering]
---

# Hot-Cold Tiering

## 定义

根据数据访问热度或时间新旧，将热数据放在高速介质、冷数据放在低成本介质的分层存储策略。

## 核心要点

1. 以 segment 等迁移单元记录热度，避免逐记录管理开销。
2. SSD 提供低延迟与高 IOPS，HDD 提供低容量成本。
3. 目标冷数据比例控制成本—性能权衡。
4. 迁移通常与 compaction / GC 合并，减少额外写放大。

## 代表工作

- [[TokaDB]]：为每个用户配置 $F_{HDD}$，把较老行为 segment 放入 HDD；$F_{HDD}\le 0.7$ 时读吞吐基本稳定。

## 相关概念

- [[Detached-Segment Index Layout]]
- [[Locality-Triggered Garbage Collection]]
- [[SSD]]
- [[HDD]]
