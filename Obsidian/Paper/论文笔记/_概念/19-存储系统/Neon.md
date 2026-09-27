---
type: concept
aliases: [Neon Database, Neon Postgres]
---

# Neon

## 定义

开源的 **serverless PostgreSQL** 项目，其核心创新是把 PostgreSQL 的存储层拆成 [[Safekeeper]]（WAL 共识复制）与 [[Pageserver]]（页物化），数据落在[[Object Storage|对象存储]]，从而支持 [[Scale-to-Zero]]、[[Database Branching|零拷贝分支]]与亚秒启动。Databricks 将 Neon 团队与技术吸收后构建了 [[Lakebase]]。

## 核心要点

1. **架构二拆分**：计算无状态化，WAL 由 SK 做 quorum 复制，页由 PS 物化并提供 `GetPage@LSN` / `GetBaseBackup` 两个 API。
2. **分支即 timeline**：项目（tenant）下可有多条 [[Timeline]]，共享 [[Pageserver]] 并按 copy-on-write 递归读父 timeline。
3. **[[NeonVM]]**：在 Kubernetes 中运行 VM 的 API，支持原地垂直扩容与热迁移，是 serverless 计算层的底座。
4. **与 [[Lakebase]] 的关系**：Lakebase 论文致谢明确感谢"跨 Databricks 与 Neon 的工程师、经理和产品团队"以及"开源 Neon 数据库社区，本工作建立在他们的贡献之上"。可以认为 Lakebase = Neon 的存储计算分离内核 + Databricks 的湖仓生态（UC / Delta / Spark / Lakehouse//RT）+ 面向 agent 的产品化。

## 代表工作

- [[Lakebase]]: 直接的技术前身与开源基础。

## 相关概念

- [[Safekeeper]]
- [[Pageserver]]
- [[NeonVM]]
- [[Timeline]]
- [[PostgreSQL]]
