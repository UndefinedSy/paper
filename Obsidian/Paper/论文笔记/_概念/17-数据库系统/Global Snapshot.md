---
type: concept
aliases: [Global Snapshot Message, 全局快照, 全局快照消息]
---

# Global Snapshot

## 定义

Firestore Frontend 发送给客户端的协议消息，表示该连接上**所有**活跃实时查询都已推进到某个特定的提交版本，用于保证同一连接上多条查询的跨查询强一致视图。

## 核心要点

1. **跨查询一致性保证**：同一连接上的多条实时查询之间，用户看到的结果永远来自同一时间戳的视图，避免「加入了聊天室 C 但 C 还没出现在房间列表里」类的不一致。
2. **触发条件**：当查询所涉及的所有分片的 [[Safetime]] 最小公共提交版本越过某一阶段时，Frontend 发出全局快照消息。
3. **协议结构**：全局快照之间穿插着「自上次快照以来各文档的最新状态」以及「变更集合」，包含新加入 / 已修改 / 已删除的文档描述。
4. **SDK 侧的缓存更新**：客户端在每次全局快照时把 zone 1（已确认服务端文档区）更新为该快照时间点的一致视图，并通知应用层所有活跃查询的最新结果集。
5. **轮询模式下的辅助信息**：快照消息中还可携带结果总数与结果文档键的 [[Bloom Filter]]，供 SDK 在无法收到显式删除通知时校验是否存在文档离开结果集。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[Safetime]]
- [[Frontend]]
- [[Polling Mode]]
- [[Streaming Mode]]
- [[Bloom Filter]]
- [[Realtime Query]]
