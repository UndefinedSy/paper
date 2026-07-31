---
type: concept
aliases: [Vitess]
---
# Vitess
## 定义
由 YouTube 开发的 MySQL 分片中间件与集群管理系统（Go 实现），后成为 CNCF 项目。
## 核心要点
1. 在 MySQL 之上提供水平分片、连接池与查询路由。
2. 被 [[Aurora Limitless Database]] 归为「提供水平扩展但**运维管理复杂**」一类。
3. 与数据库内建透明分片的差异在于：中间件路线需要额外的拓扑与元数据运维体系。
## 代表工作
- Vitess: Scaling MySQL at YouTube using Go (LISA 2012)。
## 相关概念
- [[Sharding]]
- [[Apache ShardingSphere]]
