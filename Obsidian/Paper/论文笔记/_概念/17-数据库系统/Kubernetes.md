---
type: concept
aliases: [K8s]
---

# Kubernetes

## 定义

一个开源的容器编排平台，用于自动化部署、扩展和管理容器化应用，是许多云原生数据库实现自动化故障恢复、滚动发布等运维能力的基础设施。

## 核心要点

1. 提供容器调度、自愈（自动重启失败容器）、滚动升级等能力
2. 对无状态计算节点（见 [[Separation of Compute and Storage]]）天然友好：节点失败后可被自动替换，状态从存储层恢复

## 代表工作

- [[SalesforceDB]]：所有服务运行在 Kubernetes 上，实现自动化故障恢复与服务部署；SQL 计算节点被终止后由 Kubernetes 自动拉起替代实例并从存储恢复状态

## 相关概念

- [[Separation of Compute and Storage]]
- [[SalesforceDB]]
