---
type: concept
aliases: [Google SQL, GoogleSQL dialect]
---

# GoogleSQL

## 定义

Google 内部使用的 SQL 方言，被多个查询引擎和服务共享，扩展支持了 Bigtable 的宽列数据模型查询需求。

## 核心要点

1. Google 内部多个查询引擎和服务的通用 SQL 方言
2. 为 [[Bigtable]] 扩展支持集合类型（map、set）和 temporal filter
3. 支持 `as_of` 算子进行时间点查询
4. 支持结构性变换（pivoting、unnesting）适配宽列架构
5. 支持 Protobuf descriptor，将序列化对象视为用户自定义类型
6. Row key schema 将不透明 byte-string 映射为强类型 SQL 列

## 代表工作

- [[Bigtable]]: SQL 查询接口

## 相关概念

- [[Bigtable]]
- [[Change Data Capture]]
- [[Materialized Views]]
