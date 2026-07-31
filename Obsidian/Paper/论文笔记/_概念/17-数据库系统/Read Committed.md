---
type: concept
aliases: [RC, 读已提交]
---
# Read Committed
## 定义
SQL 标准隔离级别之一：事务只能读到已提交的数据，但同一事务内的不同语句可以看到不同的数据库状态（允许不可重复读与幻读）。
## 核心要点
1. 与 [[Repeatable Read]] 的关键区别是快照粒度：RC 是**逐语句**取新快照，RR 是**整个事务**共用一个快照。
2. 在 [[Aurora Limitless Database]] 中，RC 的每条语句都重新取 $\texttt{now().latest}$ 作为快照时间戳，因此语句看到越来越新的数据。
3. Limitless 的 RC 还**关闭写冲突检测**：仍取行排他锁并持有至提交，但不检查该行是否被他人改过，事务照常继续。
4. RC 是 PostgreSQL 的默认隔离级别，也是生产中最常用的级别之一。
5. [[CockroachDB]] 支持 serializability 与 RC 但不支持 SI，属于另一种取舍。
## 代表工作
- [[AuroraLimitless]]：以逐语句时间戳 + 关闭冲突检测实现分布式 RC。
## 相关概念
- [[Repeatable Read]]
- [[Snapshot Isolation]]
- [[MVCC]]
- [[Timestamp-Based MVCC]]
