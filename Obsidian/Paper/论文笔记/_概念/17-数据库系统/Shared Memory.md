---
type: concept
aliases: [进程内共享内存通信]
---

# Shared Memory

## 定义

同一物理机内多个进程/节点通过映射同一块内存区域直接读写数据，从而避免网络连接与数据拷贝的通信机制。

## 核心要点

1. 相比网络 socket 通信，共享内存可实现零拷贝、无连接开销的数据交换。
2. [[TDSQL]] 的 [[Forward Node]] 用共享内存承载服务器内 Coordinator/Datanode 间的数据转发。
3. 常与消息队列（Fragment/Sender/Receiver Message Queue）配合，实现异步生产者-消费者模型。

## 代表工作

- [[TDSQL]]：Forward Node 服务器内通信的核心机制。

## 相关概念

- [[Forward Node]]
- [[MPP Architecture]]
