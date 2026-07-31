---
type: concept
aliases: [TPCC]
---
# TPC-C
## 定义
TPC 组织制定的经典 OLTP 基准：模拟批发商的订单处理业务，是衡量事务处理吞吐的行业标准。
## 核心要点
1. 包含 5 类事务：New-Order、Payment、Order-Status、Delivery、Stock-Level，其中 New-Order 是主要吞吐指标来源。
2. 数据规模由 warehouse 数量决定，且规定了各事务的混合比例与思考时间。
3. 官方指标是 tpmC；[[HammerDB]] 等衍生实现常用 NOPM 作为等价指标。
4. 被广泛用于分布式数据库的吞吐宣称：[[OceanBase]] 7.07 亿 tpmC、[[PolarDB]] 20 亿 tpmC 均基于它。
5. 在分片系统上运行时，如何分片 TPC-C 的表以及分布式事务占比是关键的实验设计选择。
## 代表工作
- [[HammerDB]]：TPC-C 的开源派生实现。
- [[OceanBase]] / [[PolarDB]]：以 tpmC 成绩著称的系统。
## 相关概念
- [[HammerDB]]
- [[Sharding]]
