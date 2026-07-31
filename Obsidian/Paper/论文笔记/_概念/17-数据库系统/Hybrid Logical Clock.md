---
type: concept
aliases: [HLC, 混合逻辑时钟]
---
# Hybrid Logical Clock
## 定义
把物理时钟与 Lamport 逻辑时钟结合的时钟方案：时间戳既近似真实时间（便于人类理解与外部比较），又能通过消息传递单调推进以捕获因果序。
## 核心要点
1. 典型更新规则是取「本地逻辑时钟」与「物理时钟读数」的最大值，并在必要时加 1 以保证严格单调。
2. 在 [[Aurora Limitless Database]] 中每个 shard 维护逻辑时钟 $C$：被事务 $T$ 读取时推进为 $\max\{C, T.\text{startTs}+1\}$。
3. shard 的 prepare/单分片 commit 时间戳则取 $\max\{C, \texttt{now().latest}\}$。
4. 上述两条合起来保证：一旦 $T$ 读过某 shard，该 shard 此后提交的事务 commitTs 必然 > $T.\text{startTs}$，于是读无需等待。
5. [[CockroachDB]] 与 [[YugabyteDB]] 也用 HLC 定 commit 时间戳，但在缺少高精度时钟服务时须保守地等出数百毫秒的最大偏移。
## 代表工作
- [[AuroraLimitless]]：用 HLC 消除 Clock-SI 的读延迟。
- [[CockroachDB]] / [[YugabyteDB]]：以 HLC 作为分布式事务时间戳基础。
## 相关概念
- [[Clock-SI]]
- [[Amazon Time Sync]]
- [[TrueTime]]
- [[Timestamp-Based MVCC]]
