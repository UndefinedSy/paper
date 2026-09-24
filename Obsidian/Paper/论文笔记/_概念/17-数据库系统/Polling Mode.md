---
type: concept
aliases: [Polling Mode, 轮询模式, 轮询查询]
---

# Polling Mode

## 定义

Firestore 实时查询在**流式模式**（Streaming Mode）不可用时的降级状态：通过定期向 Backend 发起普通读取查询来获取最新结果集，而非持续接收增量变更流。

## 核心要点

1. **查询生命周期起点**：每条新实时查询都从 polling mode 开始——Frontend 先向 Backend 发起一次普通查询取初始结果，之后才向 QueryMatcher 注册并切换到 streaming mode。
2. **降级触发条件**：QueryMatcher 或 ChangeLog 任务失败/重启、分片重新分配、ChangeLog 标记某时间区间不完整——这些事件均会让受影响查询退回 polling mode。
3. **恢复机制**：退回 polling mode 的查询会持续轮询直到读到的时间越过不完整区间，此后重新切换为 streaming mode；通常在一分钟内恢复。
4. **limit/offset 查询的特殊处理**：当有文档从满额结果集中离开时，Frontend 也会触发一次轮询查询来补充结果集，维持 limit 语义。
5. **Bloom filter 辅助删除检测**：在 polling mode 下发送全局快照时，附带结果总数与键的 [[Bloom Filter]]，供 SDK 判断是否有文档离开结果集，避免作废整条查询。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[Streaming Mode]]
- [[Realtime Query]]
- [[ChangeLog]]
- [[QueryMatcher]]
- [[Global Snapshot]]
- [[Bloom Filter]]
