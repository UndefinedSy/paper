---
type: concept
aliases: [外部一致性, Linearizability for Transactions]
---
# External Consistency
## 定义
一种最强的事务顺序保证：若事务 $T_2$ 在 $T_1$ 结束之后才开始提交，则 $T_2.\text{commitTs} > T_1.\text{commitTs}$，即时间戳顺序与真实时间顺序一致，从而禁止读到陈旧数据。
## 核心要点
1. 比「尊重实时序的 SI」更强：后者只约束 startTs 与 commitTs 的关系，前者约束两个 commitTs。
2. 实现通常依赖有界误差时钟 + [[Commit Wait]]。
3. [[CockroachDB]] 一般不保证该性质；[[Spanner]] 与 [[Aurora Limitless Database]] 提供。
4. 它同时也是防止 zombie 节点提供陈旧读必须守住的性质，故需要 [[Lease]] 机制配合。
5. 对客户体验的直接价值：事务总能看到最新数据，无论从哪个节点发起。
## 代表工作
- [[Spanner]]：通过 TrueTime + commit wait 实现。
- [[AuroraLimitless]]：通过 Amazon Time Sync + commit wait + 读租约实现。
## 相关概念
- [[Commit Wait]]
- [[TrueTime]]
- [[Amazon Time Sync]]
- [[Lease]]
- [[Snapshot Isolation]]
