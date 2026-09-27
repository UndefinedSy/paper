---
type: concept
aliases: [UC, Databricks Unity Catalog]
---

# Unity Catalog

## 定义

Databricks 的统一**数据治理与身份层**：集中管理认证、授权与目录元数据，跨引擎（Spark、SQL 仓库、Model Serving 等）统一访问控制。

## 核心要点

1. **身份传播**：调用者的 Databricks 有效身份经 **OAuth** 传播到 [[Lakebase]]，首次使用时映射为 [[PostgreSQL]] 角色——下游服务（Spark 管道、Model Serving）以自身身份连接，无需单独维护数据库凭证。
2. **授权下放**：Databricks 改造了 Unity Catalog，把访问控制与执行**委托给 Lakebase**，统一"直连 PostgreSQL"与"经 catalog 访问"两条路径。
3. **GRANT 翻译**：管理员在 UC 中发标准 `GRANT SELECT`，会被翻译为原生 PostgreSQL 查询并**同步**应用。
4. **消除 ACL 漂移**：UC 的目录元数据**实时**由 PostgreSQL 系统表提供，因此 UC 中的视图始终反映数据库当前状态，治理层与运行库之间不会出现权限定义漂移。
5. **与湖仓一致性的关系**：它是 Lakebase 与 Lakehouse 三个集成轴之一（统一治理 / 湖数据服务化 / 运行数据分析），价值在于让 OLTP 与湖仓共用同一套身份语义。

## 代表工作

- [[Lakebase]]: 通过 UC 完成统一认证授权，使 OLTP 直接融入湖仓生态。

## 相关概念

- [[Lakehouse]]
- [[Direct-to-Storage]]
- [[Lakebase]]
