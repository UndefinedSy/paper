---
type: concept
aliases: [计算存储分离, 存算分离]
---

# Separation of Compute and Storage

## 定义

云原生数据库架构中将无状态计算层（负责查询处理、事务执行）与持久化存储层解耦的设计原则，二者可独立伸缩、独立部署，是现代云原生 OLTP/OLAP 数据库（如 Aurora、Spanner、Socrates、Snowflake 等）的共同基础架构选择。

## 核心要点

1. 计算节点无状态，可随时被终止和替换（由编排系统如 Kubernetes 拉起新实例），状态完全来自存储层恢复
2. 存储可被多个计算实例/数据库共享，是多实例共享存储集群、实例间数据迁移等能力的前提
3. 相比传统 shared-nothing 架构，存算分离使计算和存储可以按各自的负载特征独立扩容，避免资源浪费
4. 通常需要搭配分布式缓存层以弥补存储（尤其云对象存储）与计算之间的延迟差距

## 代表工作

- [[SalesforceDB]]：无状态 SQL 计算节点从共享的 [[SDBStore]] 读取数据，多个数据库实例可共享同一存储集群，是其[[LSM Virtualization]]、租户迁移等能力的底层支撑
- [[Amazon Aurora]]：日志即数据库（log is the database）的存算分离先驱
- [[Spanner]]：全球分布式存算分离架构代表
- [[Snowflake]]：OLAP 场景下的存算分离典范，支持零拷贝克隆

## 相关概念

- [[SDBStore]]
- [[Kubernetes]]
- [[Immutability]]
