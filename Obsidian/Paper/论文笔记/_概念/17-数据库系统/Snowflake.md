---
type: concept
aliases: [Snowflake Elastic Data Warehouse]
---
# Snowflake
## 定义
云原生弹性数据仓库，以存算分离 + 多集群共享数据架构著称。
## 核心要点
1. 虚拟仓库（compute）与共享存储解耦，可独立弹性伸缩。
2. [[Aurora Limitless Database]] 称自己扩展了 Aurora 的存算分离，路线**类似 Snowflake 但面向 OLTP 优化**。
3. 定位差异：Snowflake 面向 OLAP 与大规模扫描，Limitless 面向高并发短事务。
## 代表工作
- The Snowflake elastic data warehouse (SIGMOD 2016)。
- [[TDSQL]]：作为 10TB TPC-DS 场景下的实用云仓库对比基线（6X-Large 配置）。
## 相关概念
- [[Disaggregated Storage-Compute Architecture]]
- [[Shared-Storage Architecture]]
- [[HTAP]]
