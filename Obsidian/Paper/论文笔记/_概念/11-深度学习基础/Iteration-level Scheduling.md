---
type: concept
aliases: [迭代级调度, Continuous Batching]
---

# Iteration-level Scheduling

## 定义

以一次 token 解码迭代而非整个请求为调度单位的 LLM 服务批处理机制：每轮移除完成序列、吸纳新序列，并避免将所有请求填充到相同长度。

## 核心要点

1. 新请求只需等待一个 decoding iteration，而不必等待整批完成。
2. 提升 GPU 计算利用率，但活跃 KV cache 会随请求动态进出，增加内存管理难度。
3. 与 [[PagedAttention]] 互补：前者提高时间维度的交错度，后者提高空间维度的 KV 利用率。

## 代表工作

- [[Orca]]：提出面向 Transformer 生成的 iteration-level scheduling。
- [[PagedAttention]]：在细粒度调度下以块式 KV 管理增加可并发请求数。

## 相关概念

- [[Large Language Model Serving]]
- [[KV Cache]]
- [[Preemption]]
