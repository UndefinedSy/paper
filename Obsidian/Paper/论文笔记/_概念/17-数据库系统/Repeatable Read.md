---
type: concept
aliases: [RR, 可重复读]
---
# Repeatable Read
## 定义
SQL 标准隔离级别之一：事务内对同一数据的重复读取结果一致。在 PostgreSQL 的实现中，RR 实际提供的是 [[Snapshot Isolation]]。
## 核心要点
1. 整个事务的所有语句共用同一个快照，因此读结果稳定可重复。
2. PostgreSQL 语义下 RR = SI，故仍存在 write skew 这类 SI 特有异常，并非可串行化。
3. [[Aurora Limitless Database]] 的 RR 用单一 startTs 贯穿事务，并开启写冲突检测（发现更新版本即中止）。
4. Limitless 进一步强化为「尊重实时序的 SI」乃至 [[External Consistency]]。
## 代表工作
- [[AuroraLimitless]]：把 PostgreSQL 的 RR 语义扩展到跨分片场景。
## 相关概念
- [[Snapshot Isolation]]
- [[Read Committed]]
- [[Serializability]]
- [[Timestamp-Based MVCC]]
