---
type: concept
aliases: [FDB, 基础数据库]
---
# FoundationDB

## 定义
FoundationDB 是一个开源的**分布式非捆绑（unbundled）事务性键值存储**（SIGMOD 2021），把事务日志（TLog）、存储节点与事务层拆成可独立扩展的角色，同时对外提供可串行化的键值事务接口。

## 核心要点
1. **非捆绑架构**：TLog 与存储节点解耦，存储层异步拉取并应用日志；另有 commit proxy、resolver、master/data distributor 等角色。
2. **全局提交序**：commit proxy 与 resolver、TLog 协同强制全局提交顺序——**这正是它在压力测试下的瓶颈所在**（commit proxy 可成为瓶颈）。
3. **与 [[K2Plus]] 的对比**：两者都把日志与存储解耦，但 FDB 仍依赖全局提交序与 commit proxy；K2+ 用 [[TrueTime]] 排序，进一步把事务层与复制层**完全**解耦，且不需要 per-range 共识。
4. **实测数据**：多区域同步复制 TPC-C 中 FDB v6.0 表现优于 [[TiDB]]，但落后于 K2+；App#2 单区域场景中 FDB 为 150.19 Ktps，K2+ 达 415.05 Ktps（2.76×）。

## 代表工作
- [[K2Plus]]：以 FoundationDB 作为主要的多区域 OLTP 对比基线之一。
- [[CockroachDB]]：另一条把 SQL 层架在分布式 KV 上的路线。

## 相关概念
- [[K2Plus]]
- [[CockroachDB]]
- [[Spanner]]
- [[Replication Middleware]]
- [[WAL]]
