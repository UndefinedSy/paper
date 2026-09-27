---
type: concept
aliases: [Microsoft Socrates, Azure SQL Database Socrates]
---

# Socrates

## 定义

微软 Azure SQL Database 的**云原生存储计算分离架构**（SIGMOD 2019）。它把日志服务、页服务与计算分离，是"第二代数据库"的代表实现之一。

## 核心要点

1. **分离但不开放**：改善弹性与可用性，但存储格式仍是**厂商私有**、数据只能通过主数据库引擎访问——这是 [[Lakebase]] 把 Gen-2 与 Gen-3 划开的关键判据。
2. **与同代系统并列**：论文常与 [[Amazon Aurora]]、AlloyDB、SolarDB、Taurus、Tell 并列为 Gen-2 的样本。
3. **局限**：多引擎访问需要数据复制（ETL），跨云可移植性受限，迁移成本高——即 [[Lakebase]] 所称的"结构性厂商锁定"。

## 代表工作

- [[Amazon Aurora]]: 同代系统。
- [[Lakebase]]: 论文将 Socrates 归为 Gen-2，作为对比的架构原型之一。

## 相关概念

- [[Amazon Aurora]]
- [[Disaggregated Storage-Compute Architecture]]
- [[Lakebase]]
- [[PostgreSQL]]
