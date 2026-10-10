---
type: concept
aliases: [Key Management System, 密钥管理系统]
---

# KMS

## 定义

Key Management System，负责生成、分发、轮换和撤销加密密钥的系统组件，是实现数据加密（尤其是细粒度加密如按租户加密）不可或缺的基础设施。

## 核心要点

1. 管理加密密钥的全生命周期，支持密钥轮换（rotation）
2. 在 SalesforceDB 中，KMS 负责管理 [[Tenant Encryption]] 所需的按租户加密密钥，密钥轮换通过触发 LSM compaction 完成数据的异步重加密

## 代表工作

- [[SalesforceDB]]：Salesforce 内部 KMS 负责管理超过 7,000 个租户的独立加密密钥

## 相关概念

- [[Tenant Encryption]]
- [[LSM-tree]]
