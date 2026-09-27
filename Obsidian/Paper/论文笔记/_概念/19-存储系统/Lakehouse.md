---
type: concept
aliases: [湖仓一体, Lakehouse]
---

# Lakehouse

## 定义

一种把**数据仓库**的结构化事务能力与**数据湖**的低成本开放存储合并的架构范式（Armbrust et al., CIDR 2021）。数据以开放格式（Parquet / Delta / Iceberg 等）存在对象存储上，由多种引擎共享访问，避免传统"湖 + 仓"两套系统的数据复制。

## 核心要点

1. **开放格式 + 共享存储**：数据只用开放格式落一次，多引擎（Spark、实时分析引擎、ML 训练）直接读取，不经过 ETL 复制。
2. **与 OLTP 的张力**：传统上湖仓服务分析负载，而事务负载落在私有格式的数据库中——两者之间的数据搬运（ETL）是主要摩擦点。
3. **[[Lakebase]] 的三条集成通路**：
   - **统一治理**：通过 [[Unity Catalog]] 共享身份与授权。
   - **湖数据服务化**（Lakehouse → OLTP）：Synced Tables，含标准 Spark 流路径与 [[Direct-to-Storage]] 大快照路径。
   - **运行数据分析**（OLTP → 分析）：`wal2delta` 逻辑解码把变更以 SCD Type 2 追加到 Delta 表；以及 Direct Access 让 Spark / Lakehouse//RT 直接读 PostgreSQL 页文件。
4. **`Lakehouse//RT`**：Databricks 的实时分析引擎，用 [[SIMD]] 解析 PostgreSQL 页 + 简化 [[MVCC]] 检查，缓存 Arrow 批次，并用 [[Roaring Bitmap|modified-blocks bitmap]] 只刷新过期页。TPC-H SF=10 上比 PostgreSQL 总时长低 10×、几何平均延迟改善 3×。
5. **数据不动、算力动**：与"把数据搬到分析系统"相反，湖仓式集成让分析引擎去读 OLTP 的开放存储，避免持久化的列存副本。

## 代表工作

- [[Lakebase]]: 把 OLTP 直接放在 Lakehouse 旁边，共享同一份对象存储。
- [[Snowflake Unistore]] / Microsoft Fabric: 同样意图融合事务与分析，但 OLTP 侧格式仍私有。

## 相关概念

- [[Unity Catalog]]
- [[Direct-to-Storage]]
- [[Object Storage]]
- [[Apache Arrow]]
- [[HTAP]]
