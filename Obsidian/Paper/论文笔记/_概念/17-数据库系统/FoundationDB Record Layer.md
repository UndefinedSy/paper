---
type: concept
aliases: [Record Layer]
---

# FoundationDB Record Layer

## 定义

Apple 构建在 FoundationDB 之上的多租户结构化数据存储层，通过逻辑上的租户作用域子空间（tenant-scoped subspace）实现按租户聚簇数据。

## 核心要点

1. 同样实现了"按租户聚簇数据"的效果，但这是通过在存储层之上构建逻辑子空间实现的，而非像 SalesforceDB 那样把多租户语义直接嵌入存储引擎本身
2. 这一架构差异决定了两者在租户迁移、加密等能力上的实现方式不同：SalesforceDB 可以在存储引擎层做租户级元数据操作（如 [[LSM Virtualization]]），而 Record Layer 的租户隔离主要是逻辑层面的命名空间划分

## 代表工作

- FoundationDB Record Layer 论文（Chrysafis et al., 2019, SIGMOD）

## 相关概念

- [[SalesforceDB]]
- [[LSM Virtualization]]
- [[FoundationDB]]
