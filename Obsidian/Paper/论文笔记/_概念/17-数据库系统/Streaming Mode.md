---
type: concept
aliases: [Streaming Mode, 流式模式, 流推送模式]
---

# Streaming Mode

## 定义

Firestore 实时查询在初始化完成后进入的主要工作状态：[[ChangeLog]] 通过 [[QueryMatcher]] 向 [[Frontend]] 持续推送文档变更，Frontend 再以流的形式实时通知客户端，而无需客户端反复轮询。

## 核心要点

1. **从 polling 切换而来**：每条新实时查询都从 [[Polling Mode]] 开始（取初始结果集），完成后切换到 streaming mode；切换延迟 p50 约 130–260ms（随结果集规模增长）。
2. **增量更新**：只推送自上次 [[Global Snapshot]] 以来变更的文档（新增、修改、删除），而非全量结果集，大幅减少带宽消耗。
3. **降级触发**：QueryMatcher 或 ChangeLog 任务失败/重启、分片重新分配、变更流不完整时，受影响查询退回 polling mode，通常在一分钟内恢复 streaming。
4. **扩展性来源**：streaming mode 的读扩展依靠 Frontend 横向扩展和多 QueryMatcher 之间的连接负载均衡，与写扩展（分片数量）相互独立。
5. **延迟来源**：commit 到消息抵达 Frontend p50 ~21ms，再到 Global Snapshot 发出 p50 ~15ms，整体通知延迟 p50 ~40ms（Table 1）。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[Polling Mode]]
- [[Realtime Query]]
- [[ChangeLog]]
- [[QueryMatcher]]
- [[Frontend]]
- [[Global Snapshot]]
- [[Safetime]]
