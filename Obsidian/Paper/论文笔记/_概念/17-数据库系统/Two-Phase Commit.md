---
type: concept
aliases: [2PC, 两阶段提交]
---
# Two-Phase Commit
## 定义
分布式事务的经典原子提交协议：coordinator 先向所有参与者发 prepare 征询，收齐同意后再发 commit，从而保证所有参与者对事务结局达成一致。
## 核心要点
1. prepare 阶段参与者须把「我能提交」这一决定持久化，此后不得反悔。
2. 经典 2PC 的致命弱点是 coordinator 故障会让已 prepare 的参与者无限期阻塞（blocking）。
3. 常见解法是给 coordinator 加共识复制（Paxos/Raft），代价是复杂度与延迟。
4. [[Aurora Limitless Database]] 的做法是把状态持久化到有 standby 的 [[Lead Shard]]，用最小成本换到秒级恢复。
5. 只读事务与单分片写事务可完全绕过 2PC，这是生产系统最重要的优化。
## 代表工作
- [[AuroraLimitless]]：lead shard 变体 + 时间戳集成，容忍 router 故障。
- [[Citus]]：用 2PC 保证 A/C/D，但不提供分布式 SI。
- [[Spanner]]：2PC 的每个参与者组内部再用 Paxos 复制。
## 相关概念
- [[Lead Shard]]
- [[Snapshot Isolation]]
- [[Commit Wait]]
- [[Sharding]]
