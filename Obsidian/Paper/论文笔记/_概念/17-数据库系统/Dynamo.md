---
type: concept
aliases: [Amazon Dynamo]
---
# Dynamo
## 定义
Amazon 的高可用键值存储，以最终一致性、一致性哈希与 quorum 复制奠定了 NoSQL 的设计范式。
## 核心要点
1. 一致性哈希 + 虚拟节点做数据放置与负载均衡。
2. sloppy quorum + hinted handoff 换取写可用性；用 vector clock 做版本调和。
3. 被 [[Aurora Limitless Database]] 列为其查询处理与分布式设计所借鉴的经典研究之一。
## 代表工作
- Dynamo: Amazon's highly available key-value store (SOSP 2007)。
## 相关概念
- [[DynamoDB]]
- [[Eventual Consistency]]
- [[CRDT]]
