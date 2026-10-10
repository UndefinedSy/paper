---
type: concept
aliases: [CockroachDB Serverless]
---

# CockroachDB Serverless

## 定义

[[CockroachDB]] 的无服务器化产品形态，将数据库节点解耦为每租户独立的 SQL 层与共享的、以 tenant-id 前缀 key 隔离的 shared-process KV 层。

## 核心要点

1. 采用 per-tenant SQL 层 + shared-process KV 层的架构，KV 层通过 tenant-id key 前缀实现租户间数据隔离
2. 与 SalesforceDB 的关键差异：CockroachDB Serverless 的租户之间**不共享同一套 schema**，而 SalesforceDB 的所有租户共享同一物理 schema，这使得 SalesforceDB 能够支持跨数据库实例的租户迁移（依赖 schema fingerprint 比对），而 CockroachDB Serverless 难以做到类似的跨库零拷贝迁移
3. 支持从零弹性伸缩（sub-second scaling from zero）

## 代表工作

- CockroachDB Serverless 论文（Swenson et al., 2025, SIGMOD Companion）

## 相关概念

- [[CockroachDB]]
- [[SalesforceDB]]
- [[Shared Memory]]
