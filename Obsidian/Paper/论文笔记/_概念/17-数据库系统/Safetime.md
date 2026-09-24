---
type: concept
aliases: [Safe Time, 安全时间, 安全版本]
---

# Safetime

## 定义

Firestore 实时栈中表示「某分片范围上直到该版本的所有变更均已被接收」的最高提交版本，是判定变更能否安全释放给客户端以及变更流是否完整的核心量。

## 数学形式

$$
\text{safetime} = \max \left\{ v \;\middle|\; \text{分片范围 } r \text{ 上直到提交版本 } v \text{ 的所有变更均已接收} \right\}
$$

跨多分片查询的全局安全时间取各分片的最小值：

$$
\text{global\_safetime} = \min_i \, \text{safetime}_i
$$

## 核心要点

1. **三类消息驱动**：心跳消息（空闲时定期推进）、blocked shard 消息（有在途事务时压制推进，取下界）、unblocked shard 消息（事务完成后恢复推进）。
2. **控制变更释放时机**：Frontend 只有在某变更的提交时间 $v \le \text{global\_safetime}$ 时才向客户端发送该变更，确保不将无法保证完整性的变更提前曝光。
3. **不完整区间的边界**：当 Accept 超时或分片所有权转移时，ChangeLog 把 [lower\_bound, upper\_bound]（由 [[TrueTime]] 决定）标记为不完整，使 safetime 无法越过该区间，上游据此降级受影响查询。
4. **时间分辨率来自 TrueTime**：提交时间的上下界由 [[TrueTime]] 调用确定，Prepare 阶段的下界就是 blocked shard 消息的 safetime 压制值。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[ChangeLog]]
- [[TrueTime]]
- [[Global Snapshot]]
- [[Polling Mode]]
- [[Streaming Mode]]
