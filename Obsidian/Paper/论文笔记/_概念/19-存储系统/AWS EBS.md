---
type: concept
aliases: [Elastic Block Store]
---

# AWS EBS

## 定义

Amazon Elastic Block Store，AWS 提供的持久化块存储服务，为 EC2 实例提供网络附加的块设备，常用于需要低延迟持久写入的场景（如事务日志）。

## 核心要点

1. 相比 S3 等对象存储，EBS 提供更低延迟的块级读写，适合作为日志（WAL）等需要持久且快速写入的数据的后端
2. 在云原生数据库架构中，常与对象存储搭配使用：EBS 做低延迟热数据（如事务日志缓存），对象存储做长期持久化

## 代表工作

- [[SalesforceDB]]：[[SDBStore]] 的事务日志缓存（write-back log cache）基于 AWS EBS 类持久块存储实现，提供 1–3ms 的低延迟持久写

## 相关概念

- [[SDBStore]]
- [[AWS S3]]
