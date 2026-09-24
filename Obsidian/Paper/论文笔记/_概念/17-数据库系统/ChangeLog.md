---
type: concept
aliases: [Changelog, Change Log, 变更日志组件]
---

# ChangeLog

## 定义

Firestore 实时栈的核心组件，负责把 Spanner 上已提交的写操作转换为有序的变更流，供 QueryMatcher 消费。ChangeLog **不持久化**任何日志，而是通过「变更消息 + 时间推进消息（心跳 / blocked / unblocked）」让上游组件重建出强一致的变更序列。

## 核心要点

1. **两阶段 Prepare/Accept 协议**：在 Backend 提交前向 ChangeLog 发 Prepare，获取提交时间上下界；提交后发 Accept（含 before/after 文档快照与实际提交时间）。这不是传统 2PC——Spanner 事务无论 ChangeLog 是否收到 Accept 都不会回滚。
2. **Safetime 推进机制**：空闲时发心跳推进 [[Safetime]]；有在途事务时发 blocked shard 消息压制 safetime 推进；Accept 后发 unblocked shard 消息恢复推进。
3. **不完整流的显式标记**：Accept 超时、任务崩溃或分片所有权转移时，ChangeLog 把对应时间区间标记为不完整，上游据此让受影响查询退回 polling mode 恢复。
4. **副本组 + Chubby 选主**：ChangeLog 任务按跨可用区副本组部署，组内通过 [[Chubby]] 选主（p50 ~100ms 单 region），主故障后 Backend 通常能直接重试两阶段协议完成写入。
5. **跨分片事务并行 Prepare**：涉及多个分片时，Backend 并行与多个 ChangeLog 交互，全部 Prepare 成功后才提交。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)
- [[Firestore|Firestore (ICDE 2023)]]: Firestore: The NoSQL Serverless Database for the Application Developer

## 相关概念

- [[Safetime]]
- [[QueryMatcher]]
- [[Spanner]]
- [[TrueTime]]
- [[Chubby]]
- [[Slicer]]
- [[Two-Phase Commit]]
