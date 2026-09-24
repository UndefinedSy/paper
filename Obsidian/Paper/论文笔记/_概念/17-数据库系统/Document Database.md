---
type: concept
aliases: [Document Database, Document-Oriented Database, 文档数据库, 文档型数据库]
---

# Document Database

## 定义

以半结构化文档（通常为 JSON/BSON 等键值映射）作为基本存储单元的 NoSQL 数据库类型，每条记录是一个自描述的文档，支持嵌套结构和丰富的字段类型，不要求固定的表结构。

## 核心要点

1. **文档作为原子单位**：一个文档是字段到值的映射，值可以是基本类型、嵌套文档或数组，不需要提前定义 schema。
2. **集合组织文档**：文档按集合（collection）组织，集合之间可以层级嵌套（如 Firestore 的集合 → 文档 → 子集合）。
3. **灵活的查询**：支持按字段值过滤、排序、limit，但通常不支持多集合 join 或复杂聚合——这是为低延迟查询能力与扩展性服务的取舍。
4. **与关系型数据库的对比**：文档数据库用文档嵌套代替外键 join，适合层级化、变化快的业务对象；牺牲部分关系完整性换取模型灵活性与水平扩展能力。
5. **Firestore 的扩展**：Firestore 在文档模型上增加了 ACID 事务、强一致性和实时查询，超出了传统文档数据库的定位。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)
- [[MongoDB]]: 最广泛采用的开源文档数据库

## 相关概念

- [[Realtime Query]]
- [[Collection Group Query]]
- [[ACID]]
- [[MongoDB]]
