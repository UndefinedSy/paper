---
type: concept
aliases: [Query Matcher, 查询匹配器]
---

# QueryMatcher

## 定义

Firestore 实时栈的中间层组件，从 [[ChangeLog]] 接收文档变更流，通过 [[ChangeStream Filter]] 把每条变更匹配到订阅了相关字段的活跃实时查询，并将匹配结果转发给 [[Frontend]]。

## 核心要点

1. **O(matches) + O(log queries) 匹配代价**：依靠 [[ChangeStream Filter]]（字段列表 → [[Interval Tree]] 的 [[Trie]] 结构）避免遍历所有查询，代价与实际命中数及对数查询数相关，而非总查询数。
2. **pre/post-image 差集**：对每条变更的前后两个文档镜像各执行一次匹配，取差集得到「新命中」与「不再命中」的查询集合，支持删除类变更的高效路由。
3. **不完整流的传播**：若 ChangeLog 标记某时间区间不完整（任务超时、失败或分片所有权转移），QueryMatcher 拒绝对应查询，由 Frontend 让这些查询退回 polling mode 恢复。
4. **多租户组织**：同一 QueryMatcher 任务处理多个数据库的查询；组内静态配置 ChangeLog:QueryMatcher = 1:3（按生产需求），支持自定义扇出以应对异常负载。
5. **从 ChangeLog 消费 [[Safetime]]**：心跳、blocked shard、unblocked shard 消息通过 QueryMatcher 流转到 Frontend，驱动跨分片的安全时间推进。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[ChangeLog]]
- [[ChangeStream Filter]]
- [[Frontend]]
- [[Safetime]]
- [[Interval Tree]]
- [[Trie]]
- [[Polling Mode]]
- [[Streaming Mode]]
