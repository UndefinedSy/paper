---
type: concept
aliases: [Collection Group Query, 集合组查询]
---

# Collection Group Query

## 定义

Firestore 中允许在所有同名集合上执行单一查询的能力，用 `/**/` 语法表示，例如 `select * from /**/messages` 会跨数据库中所有名为 `messages` 的集合读取文档，无论其嵌套层级。

## 核心要点

1. **跨层级查询**：突破 Firestore 文档层级树的限制，让「查询所有聊天室中提及了我的消息」这类需求无需逐一枚举每个集合路径。
2. **典型场景**：`select * from /**/messages where id in mentions and time > T order by time desc` 可实现全局 @ 提及通知，无论消息属于哪个聊天室。
3. **分片复杂度**：集合组查询的 keyspace 可能跨越多个 ChangeLog 分片，Frontend 需要在多个 QueryMatcher 上注册该查询并维护跨分片的 [[Safetime]] 一致性。
4. **查询能力上限**：集合组查询与单集合查询共享同一套过滤/排序/limit 限制（不支持 join、聚合、否定），以维持增量可计算性。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[Realtime Query]]
- [[Document Database]]
- [[Safetime]]
