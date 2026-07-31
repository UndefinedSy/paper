---
type: concept
aliases: [DSQL]
---
# Aurora DSQL
## 定义
AWS 的 serverless 分布式 SQL 数据库，支持多 region 读写。
## 核心要点
1. 用**乐观并发控制（OCC）**提供 [[Snapshot Isolation]]，与 [[Aurora Limitless Database]] 的悲观行锁 + 2PC 路线形成对照。
2. 定位是跨 region；Limitless 定位是跨 AZ 且保持 PostgreSQL 完整兼容性。
3. 两者是 AWS 数据库产品线中面向不同伸缩维度的同门方案。
## 代表工作
- Aurora DSQL（AWS 产品）。
## 相关概念
- [[Snapshot Isolation]]
- [[Amazon Aurora]]
- [[AuroraLimitless]]
