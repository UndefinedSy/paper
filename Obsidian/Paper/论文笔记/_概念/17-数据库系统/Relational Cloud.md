---
type: concept
aliases: [RelationalCloud]
---

# Relational Cloud

## 定义

一种面向云的数据库即服务（Database-as-a-Service）架构提案（Curino et al., 2011），采用共享进程（shared-process）多租户模型，多个租户共享同一数据库服务器实例。

## 核心要点

1. 属于 shared-process 多租户模型：多租户共享数据库服务器进程，但每个租户仍可能有独立的 schema/数据库实例概念
2. 与 SalesforceDB 采用的 shared-table 模型不同，shared-process 模型的租户隔离粒度更粗，资源池化效率通常低于 shared-table

## 代表工作

- 本身是开创性提案，SalesforceDB 论文将其与 [[Elastras]] 并列为 shared-process 多租户模型的代表，并与自身的 shared-table 设计做对比

## 相关概念

- [[Elastras]]
- [[Shared Memory]]
- [[SalesforceDB]]
