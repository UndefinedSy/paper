---
type: concept
aliases: [Point-in-Time Recovery, 时间点恢复, 按时间点恢复]
---

# PITR (Point-in-Time Recovery)

## 定义

把数据库恢复到历史任意时间点（在保留窗口内）的能力。在 [[Neon]] / [[Lakebase]] 中它不是额外的备份机制，而是 [[Pageserver]] 二维层文件格式的**直接副产品**：任何窗口内的 [[LSN]] 都能被物化。

## 核心要点

1. **窗口 $P$**：超出 PITR 窗口的历史成为可回收对象，由后台垃圾回收清除。
2. **空间代价**：保留窗口内历史是空间放大的主导项 $wP/N$（$w$ 为 WAL 写入速率，$N$ 为活跃数据集大小）。Lakebase 把空间放大定义为 PS 总存储 / 用户逻辑库大小，因此天然是高估——因为系统**有义务**保留 PITR 历史且对该历史计费。生产实测约 7×。
3. **与其他机制的复用**：
   - [[Database Branching]] 的 branch LSN 被限制在父分支的 PITR 窗口内。
   - **static replica** 钉在窗口内某个 LSN 上，用 lease 机制阻止 PS 回收它需要的版本。
4. **运维价值**：分支 + PITR 取代了传统备份/恢复流程，是"降低运维复杂度"这条设计目标的实现路径。

## 代表工作

- [[Lakebase]]: PITR 窗口是分支、时间旅行查询、GC 三者的共同边界参数。
- [[Neon]]: 同一机制。

## 相关概念

- [[LSN]]
- [[Database Branching]]
- [[Pageserver]]
- [[Timeline]]
- [[Layer File]]
