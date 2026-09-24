---
type: concept
aliases: [Firebase Realtime Database, Firebase RTDB, 实时数据库]
---

# Firebase Realtime Database

## 定义

Google 在 2014 年收购 Firebase 时随之获得的实时数据库产品，以 JSON 树为数据模型、以 WebSocket 长连接为传输层，为移动和 Web 应用提供实时数据同步能力，是 Firestore 实时查询设计的直接灵感来源。

## 核心要点

1. **实时同步先驱**：Firebase RTDB 在 Firestore 之前就建立了「客户端直连数据库、变更推送到所有监听者」的编程模型，极大简化了移动应用开发。
2. **单节点变更流的限制**：与 RethinkDB、Meteor 类似，RTDB 的变更流架构在高写入负载下存在扩展瓶颈。
3. **演进为 Firestore**：Google 在 2014 年收购 Firebase 后立即开始研发 Firestore，后者在 RTDB 的编程模型基础上加入了强一致性、自动分片扩展、结构化文档模型与 ACID 事务，并于 2019 年正式 GA。
4. **影响 Firestore 的设计**：Firestore 的 SDK 离线缓存、`onSnapshot` 回调 API、以及「查询监听」的用户心智模型都直接传承自 RTDB。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)
- [[Firestore|Firestore (ICDE 2023)]]

## 相关概念

- [[Realtime Query]]
- [[Document Database]]
- [[Firestore]]
