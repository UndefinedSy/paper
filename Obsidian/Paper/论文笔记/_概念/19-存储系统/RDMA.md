---
type: concept
aliases: [RDMA, Remote Direct Memory Access]
---

# RDMA

## 定义

RDMA（Remote Direct Memory Access）是一种允许网络中一端直接读写另一端内存、绕过远端 CPU 和内核网络栈的通信技术，提供低延迟、高吞吐、低 CPU 开销的数据传输。

## 核心要点

1. **One-sided 操作**：发起方可直接对远端内存执行 read/write，无需远端 CPU 参与数据搬运。
2. **RDMA-based RPC**：常用于控制平面的小消息请求/响应（如地址查找、锁管理），可在数十微秒级完成。
3. **大数据传输**：对大块数据（如 KV cache 张量）优先用 one-sided RDMA read/write，绕开 RPC 处理开销。
4. 是构建 [[Disaggregated Memory Pool|分布式内存池]] 的基础通信层。

## 代表工作

- [[PolarKV]]：RPC（地址查找、分配、锁管理）和大数据传输（KV 张量读写）均基于 RDMA；manager 不在数据传输路径上，只处理元数据 RPC。

## 相关概念

- [[Disaggregated Memory Pool]]
- [[KV Cache Manager]]
