---
type: concept
aliases: [Realtime Queries, 实时查询, 实时监听]
---

# Realtime Query

## 定义

数据库查询的一种执行模式：在返回初始结果集之后，持续以流的形式向客户端推送该结果集的变更（新增、修改、删除），使客户端始终持有最新的一致视图，而无需轮询。

## 核心要点

1. **两阶段执行**：先以 polling 模式取初始结果（点查），然后切换为 streaming 模式持续接收增量变更。
2. **强一致性要求**：同一连接上多条实时查询必须呈现同一时间戳的视图（[[Global Snapshot]]），不能出现「加入了聊天室 C 但 C 还没出现在房间列表里」的不一致。
3. **低延迟**：变更从提交到推送到客户端的目标延迟 O(100ms)（Firestore 生产 p50 约 40ms）。
4. **查询能力受限**：为保证增量可计算，Firestore 的实时查询只支持字段与常量的等值/`in` 过滤、至多一个关系运算符、合取/析取、排序、limit/offset；不支持 join、聚合、否定、算术。
5. **与物化视图的异同**：实时查询不属于任何事务作用域，允许跳过部分更新后从当前时间重跑，不必与库中其他视图保持事务一致；物化视图则必须与数据库状态保持同步。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)
- [[Firebase Realtime Database]]: 设计灵感来源
- [[InvaliDB]]: 最接近的学术方案（最终一致）
- [[RethinkDB]]: 开源实时查询实现（单节点变更流，写扩展受限）

## 相关概念

- [[ChangeLog]]
- [[QueryMatcher]]
- [[ChangeStream Filter]]
- [[Global Snapshot]]
- [[Safetime]]
- [[Polling Mode]]
- [[Streaming Mode]]
- [[Continuous Query]]
- [[Materialized Views]]
- [[Incremental View Maintenance]]
