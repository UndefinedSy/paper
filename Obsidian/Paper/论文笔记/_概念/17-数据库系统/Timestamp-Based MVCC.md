---
type: concept
aliases: [时间戳多版本并发控制, 标量时间戳快照]
---
# Timestamp-Based MVCC
## 定义
用标量时间戳而非「运行中事务 ID 集合」来定义事务快照与版本可见性的 [[MVCC]] 变体。
## 核心要点
1. 可见性判据极简：行版本对 $T$ 可见当且仅当 $\text{xmin.commitTs} \le T.\text{startTs} < \text{xmax.commitTs}$。
2. 相比 PostgreSQL 原生的 (xmin, xmax, xip_list) 三元组快照，省掉了「记录所有运行中事务」这一 CPU 密集步骤，消除了高吞吐下的著名争用点。
3. 更关键的是它天然适配分布式：单个节点无需知晓全局运行中事务集合，只需比较标量。
4. [[Aurora Limitless Database]] 为不改动 PostgreSQL 行格式，额外维护「事务 id → commitTs」映射并塞进已有的 commit log 结构。
5. 副作用红利：提交顺序与可见顺序被同一个时间戳定义，因此二者天然一致——这正是社区版 PostgreSQL 所缺乏的性质。
## 代表工作
- [[AuroraLimitless]]：替换 PostgreSQL 快照机制的核心改造。
- [[Spanner]] / [[CockroachDB]] / [[YugabyteDB]]：均采用时间戳化的多版本可见性。
## 相关概念
- [[MVCC]]
- [[Snapshot Isolation]]
- [[Clock-SI]]
- [[Hybrid Logical Clock]]
- [[Repeatable Read]]
