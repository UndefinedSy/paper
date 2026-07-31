---
type: concept
aliases: [可串行化, SSI, Serializable Snapshot Isolation]
---
# Serializability
## 定义
最强的 SQL 隔离级别：并发事务的执行效果等价于某个串行执行顺序，排除包括 write skew 在内的所有异常。
## 核心要点
1. SI 与可串行化的差距集中在 write skew 与 read-only anomaly。
2. PostgreSQL 用 SSI（Serializable Snapshot Isolation）实现，在 SI 之上增加读写依赖追踪。
3. [[Aurora Limitless Database]] **主动放弃**了可串行化，理由有两条：客户需求有限；且 PostgreSQL SSI 实现中**序列化顺序与提交顺序不一致**，与 Limitless「用时间戳统一定义提交序与可见序」的设计哲学冲突。
4. [[CockroachDB]] 与 [[YugabyteDB]] 均支持可串行化，构成与 Limitless 的一个显著取舍差异。
## 代表工作
- [[AuroraLimitless]]：显式讨论了不支持它的理由。
- [[CockroachDB]] / [[YugabyteDB]]：支持可串行化的分布式 SQL 系统。
## 相关概念
- [[Snapshot Isolation]]
- [[Repeatable Read]]
- [[MVCC]]
