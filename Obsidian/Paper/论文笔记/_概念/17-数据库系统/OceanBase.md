---
type: concept
aliases: [OB]
---
# OceanBase
## 定义
蚂蚁集团的分布式关系数据库，以 7.07 亿 tpmC 的 TPC-C 成绩著称。
## 核心要点
1. Paxos 复制 + 分区级负载均衡的大规模 OLTP 架构。
2. 被 [[Aurora Limitless Database]] 列为其查询处理所借鉴的分布式数据库研究之一。
3. 与 Limitless 的对比参照点是：两者都追求极高 OLTP 吞吐，但 OceanBase 自建共识，Limitless 复用存储层。
## 代表工作
- OceanBase: A 707 million tpmC distributed relational database system (VLDB 2022)。
## 相关概念
- [[TiDB]]
- [[Sharding]]
- [[PolarDB]]
