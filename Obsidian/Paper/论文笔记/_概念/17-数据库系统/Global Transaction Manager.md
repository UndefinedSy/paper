---
type: concept
aliases: [Global Transaction Manager, GTM]
---

# Global Transaction Manager

## 定义

分布式数据库中负责管理全局事务（分配全局事务 ID、协调跨节点提交可见性等）的中心化组件。

## 核心要点

1. 在 MPP 架构中通常与 Coordinator、Datanode 分离部署，专职处理全局事务语义正确性。
2. [[TDSQL]] 架构中的 GTM 与 Coordinator、Datanode、[[Forward Node]] 共同构成核心处理链路。

## 代表工作

- [[TDSQL]]：架构图中的核心组件之一。

## 相关概念

- [[MPP Architecture]]
