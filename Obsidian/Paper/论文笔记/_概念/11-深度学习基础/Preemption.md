---
type: concept
aliases: [抢占, Request Preemption]
---

# Preemption

## 定义

资源不足时暂停部分正在执行的生成请求，释放其 GPU 状态，并在资源可用后恢复的在线推理调度机制。

## 核心要点

1. LLM 输出长度未知，活跃 KV cache 增长可能耗尽 GPU block pool。
2. [[PagedAttention]] 采用 FCFS 服务，资源不足时优先抢占较晚到达的请求。
3. 共享块要求以 sequence group 为整体抢占，并用 [[KV Cache Swapping]] 或 [[KV Cache Recomputation]] 恢复。

## 代表工作

- [[PagedAttention]]：为块式 KV cache 设计 all-or-nothing sequence-group eviction。

## 相关概念

- [[Iteration-level Scheduling]]
- [[KV Cache]]
- [[KV Cache Swapping]]
