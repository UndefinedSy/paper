---
type: concept
aliases: [Greenplum Database]
---
# Greenplum
## 定义
基于 PostgreSQL 的大规模并行数据库，同时服务 OLTP 与 OLAP 负载。
## 核心要点
1. 用**中心化 coordinator** 分配分布式事务标识，以此提供 [[Snapshot Isolation]]。
2. 局限一：coordinator 随负载强度上升极易成为性能瓶颈。
3. 局限二：需维护「本地事务 ↔ 分布式事务」的映射来做可见性检查，既有性能开销又增加实现复杂度。
4. [[Aurora Limitless Database]] 用时间戳并发控制 + 2PC 彻底去掉了中心协调者，正是针对这两点。
## 代表工作
- Greenplum: A hybrid database for transactional and analytical workloads (SIGMOD 2021)。
## 相关概念
- [[Snapshot Isolation]]
- [[HTAP]]
- [[Sharding]]
- [[AuroraLimitless]]
