---
type: concept
aliases: [局部性触发垃圾回收, LTGC]
---

# Locality-Triggered Garbage Collection

## 定义

一种以实体级数据碎片度而非失效空间比例为触发依据的垃圾回收机制，用于把散落在多个 [[ValueLog]] 文件中的相关 segment 重新聚合。

## 数学形式

$$
\overline{LS}>\theta \quad \text{or} \quad F_{LS}>\gamma
$$

其中 $LS_k$ 是 key $k$ 横跨的 segment 数，$\overline{LS}$ 是平均值，$F_{LS}$ 是超过高阈值 $\tau$ 的 key 比例。

## 核心要点

1. 在 compaction 时在线统计实体级碎片度。
2. 触发后重写相关 segment，恢复点读所需的空间局部性。
3. 阈值越激进，局部性越好但 GC I/O 越高。
4. 与只回收过期 value 的传统 BlobDB GC 目标不同。

## 代表工作

- [[TokaDB]]：在用户行为序列上以 $(\theta,\tau)=(3,5)$ 获得较好的局部性—I/O 平衡。

## 相关概念

- [[Detached-Segment Index Layout]]
- [[ValueLog]]
- [[LSM-tree]]
