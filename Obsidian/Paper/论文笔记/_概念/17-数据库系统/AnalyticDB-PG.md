---
type: concept
aliases: [ADB-PG, AnalyticDB-PG]
---

# AnalyticDB-PG

## 定义

AnalyticDB-PG 是阿里云推出的云原生实时数据仓库，基于 [[MPP Architecture|MPP 架构]]把数据与计算分布到集群，是 [[Nova]] 向量引擎的宿主系统。

## 核心要点

1. **Coordinator 节点**：Parser（解析客户端请求）、Optimizer（生成分布式执行计划）、Query Dispatcher（任务调度与结果聚合）、Distributed Transaction Manager（DTM，提供 [[ACID]] 保证）。
2. **Worker 节点**：类似独立数据库实例，各自维护本地索引与 [[WAL]]，执行分配的查询计划。
3. 表通常按行分区并分布到各 worker。
4. 支持 [[Foreign Data Wrapper]] 等机制扩展外部数据源。

## 代表工作

- [[Nova]]: 在 ADB-PG 内实现向量引擎，注册自定义资源管理器 `RM_NOVA_ID` 接管索引 WAL 与崩溃恢复。

## 相关概念

- [[MPP Architecture]]
- [[PostgreSQL]]
- [[WAL]]
