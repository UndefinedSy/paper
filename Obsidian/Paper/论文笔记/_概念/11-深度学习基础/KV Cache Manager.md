---
type: concept
aliases: [KV 缓存管理器, KV Block Manager]
---

# KV Cache Manager

## 定义

负责为在线 LLM 推理请求分配、映射、共享、释放与迁移 [[KV Cache]] 物理块的运行时管理组件。

## 核心要点

1. 维护每条序列的 [[Block Table]]、物理块引用计数与尾块填充状态。
2. 配合 scheduler 在新 token 需要新块时分配、在请求结束时回收。
3. 过载时以 sequence group 为单位执行 [[Preemption]]、交换或重计算恢复。

## 代表工作

- [[PagedAttention]]：中央 KV cache manager 控制所有分布式 GPU worker 的共同逻辑-物理块映射。

## 相关概念

- [[PagedAttention]]
- [[KV Cache]]
- [[KV Cache Swapping]]
