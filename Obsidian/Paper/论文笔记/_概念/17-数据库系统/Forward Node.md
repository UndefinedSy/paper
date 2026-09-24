---
type: concept
aliases: [数据转发中间层, FN]
---

# Forward Node

## 定义

部署在每台服务器上的数据通信代理节点，作为传统 MPP 全连接拓扑的替代中间层，统一管理服务器内共享内存转发与跨服务器连接复用。

## 核心要点

1. 服务器内：Coordinator/Datanode 与本地 Forward Node 通过共享内存零拷贝交换数据，无需建立连接。
2. 跨服务器：所有跨机数据经本地 Forward Node 路由，FN 间做连接复用，把集群连接规模从 $O(N^2)$ 降为 $O(n\cdot(n-1))$（$n$ 为服务器数）。
3. 模块化设计包含元数据管理、分布式共享内存、Sender（Fragment Message Queue + Merge Thread + Sender Message Queue/Thread）与 Receiver 对称模块。
4. 同时承担 [[Runtime Filter]] 的 group-wise cascading 聚合与转发职责。

## 代表工作

- [[TDSQL]]：提出 Forward Node 解决传统 [[MPP Architecture]] 的连接爆炸问题。

## 相关概念

- [[MPP Architecture]]
- [[Shared Memory]]
- [[Runtime Filter]]
