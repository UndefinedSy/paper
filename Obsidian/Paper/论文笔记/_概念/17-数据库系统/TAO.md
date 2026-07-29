---
type: concept
aliases: [The Associations and Objects, Facebook TAO]
---

# TAO

## 定义

Facebook 为社交图构建的分布式数据存储系统，以 object 与 association 为核心模型，围绕高读负载的关联列表提供缓存、分片和跨地域复制。

## 核心要点

1. Object 表示实体，Association 表示带类型和时间的有向边。
2. 常见操作围绕某个 source object 的 association list 展开。
3. 通过大规模 cache tier 承载读多写少的社交图访问。
4. 采用最终一致的跨地域复制，在可用性、延迟与一致性之间取舍。

## 代表工作

- [[ByteGraph-Dione]]：同为生产社交图系统，但进一步面向同图 OLTP/OLAP、SI 和动态热点。

## 相关概念

- [[Adjacency List]]
- [[ByteGraph]]
- [[Eventual Consistency]]
