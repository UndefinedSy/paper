---
type: concept
aliases: [InvaliDB, Invali DB]
---

# InvaliDB

## 定义

一种学术提出的可扩展实时查询系统，在传统拉取式数据库之上构建推送式实时查询能力，是与 Firestore 最接近的公开学术方案（PVLDB 2020）。

## 核心要点

1. **写分片 + 查询副本**：与 Firestore 类似，通过在多个写副本上分片写流量，再跨查询副本扩展查询匹配，实现水平扩展——发布结果显示可扩展到数万次写/查询每秒。
2. **最终一致**：与 Firestore 不同，InvaliDB 的查询是**最终一致**的，不提供强一致快照保证。
3. **分片方式疑似手动**：论文未明确描述自动分片机制。
4. **相似的查询能力**：支持 filter（过滤）、布尔运算符、排序，与 Firestore 的 filter 能力接近。
5. **学术规模**：与 Firestore 的生产规模（8M 数据库、1.5B MAU）相比，InvaliDB 的验证规模差了约两个数量级。

## 代表工作

- Wolfram Wingerath, Felix Gessert, and Norbert Ritter. InvaliDB: scalable push-based real-time queries on top of pull-based databases. PVLDB 13(12), 2020.

## 相关概念

- [[Realtime Query]]
- [[Firestore]]
- [[Eventual Consistency]]
