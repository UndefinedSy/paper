---
type: concept
aliases: [RethinkDB, Rethink Database, 实时数据库]
---

# RethinkDB

## 定义

一款以「实时推送」为核心卖点的开源文档数据库，支持客户端通过 changefeed 订阅查询结果集的变更，是最早将实时查询作为一等公民对外暴露的数据库产品之一。

## 核心要点

1. **Changefeed**：RethinkDB 的标志性 API，允许客户端订阅表或查询级别的变更流，数据库在每次写入发生后主动推送变更给订阅者，无需客户端轮询。
2. **单节点变更流的扩展限制**：RethinkDB 将整个数据库的变更流汇聚到单个节点后再做分发，这一架构在写入负载较大时成为扩展瓶颈，是其主要局限。与 Firestore 按键空间分片的做法形成对比。
3. **最终一致性**：与 Firestore 的强一致快照不同，RethinkDB 不提供跨多个 changefeed 订阅的强一致性保证。
4. **项目现状**：RethinkDB 公司于 2016 年关闭，代码后由 Linux Foundation 托管并继续以开源形式维护，但商业活跃度已大幅下降。

## 代表工作

- Leif Walsh, Vyacheslav Akhmechet, Mike Glukhovsky. RethinkDB — Rethinking Database Storage. 2009.
- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[Realtime Query]]
- [[Continuous Query]]
- [[Firebase Realtime Database]]
- [[Document Database]]
