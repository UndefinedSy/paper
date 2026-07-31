---
type: concept
aliases: [Postgres, PG]
---
# PostgreSQL
## 定义
开源的关系数据库管理系统，以功能丰富、扩展性强、标准兼容性好著称，是众多分布式数据库的基座。
## 核心要点
1. 原生 [[MVCC]]：每行多版本，用 (xmin, xmax) 事务 ID 标记，快照为 (xmin, xmax, xip_list) 三元组。
2. 关键可复用机制：分区表（partitioned table）、外部表与 [[Foreign Data Wrapper]]、commit log、[[Partition Pruning]]、heavyweight lock、partition-wise join。
3. 被 [[Aurora Limitless Database]] 评价为「构建分布式数据库极其灵活的基座」，但也指出多处需现代化：schema 演进缺少版本化/canary/schema MVCC；**提交顺序与可见顺序不一致**（已在带只读副本的集群中引发异常）；SSI 的序列化顺序与提交顺序不一致。
4. 衍生的分布式系统包括 [[Citus]]、[[Greenplum]]、[[Aurora Limitless Database]]、[[YugabyteDB]]（兼容层）等。
## 代表工作
- [[AuroraLimitless]]：在其上做时间戳 MVCC 与分布式化改造。
- [[Citus]] / [[Greenplum]]：其它 PostgreSQL 系分布式方案。
## 相关概念
- [[MVCC]]
- [[Foreign Data Wrapper]]
- [[Partition Pruning]]
- [[Timestamp-Based MVCC]]
- [[Serializability]]
