---
type: concept
aliases: [提交等待]
---
# Commit Wait
## 定义
为保持事务的实时序（real-time order）而在提交路径上插入的等待：直到有界误差时钟服务返回的时间下界超过该事务的 commit 时间戳，才向客户端确认提交。
## 核心要点
1. 形式化条件为 $\texttt{now().earliest} > T.\text{commitTs}$。
2. 作用：保证「事务对客户端可见的真实时刻」一定晚于其 commit 时间戳，从而后开始的事务必然拿到更大的 startTs。
3. 这是把「时钟不确定性」转化为「一小段真实等待」的经典手段，由 [[Spanner]] 首创。
4. 等待时长量级等于时钟误差界（[[Amazon Time Sync]] 典型 <1ms）。
5. [[Aurora Limitless Database]] 让它与写 commit log 到存储并行执行，因存储写延迟通常更大，故几乎不上关键路径。
## 代表工作
- [[Spanner]]：TrueTime 上的 commit wait 原始设计。
- [[AuroraLimitless]]：与 Time Sync 集成，并与存储写并行以隐藏延迟。
## 相关概念
- [[External Consistency]]
- [[Amazon Time Sync]]
- [[TrueTime]]
- [[Snapshot Isolation]]
