---
type: concept
aliases: [Shared-Storage Architecture, 共享存储架构, Disaggregated Storage]
---

# Shared-Storage Architecture

## 定义

共享存储架构将存储层与计算层解耦，所有计算节点共享访问统一的分布式存储，实现独立扩展、弹性资源和容错。

## 核心要点

1. 存储-计算分离：存储独立扩展，避免资源耦合
2. LindormVector 只在内存保留量化向量，原始向量持久化在 LindormDFS
3. 全局共享质心：所有 region 使用同一组质心，简化管理和扩缩容
4. 批量 get 操作减少网络通信（vs DiskANN 的多轮 get）

## 代表工作

- [[LindormVector]]: 共享存储上的 IVFPQ 设计
- [[LindormTable]]: Lindorm 宽表引擎

## 相关概念

- [[LindormDFS]]
- [[LindormVector]]
