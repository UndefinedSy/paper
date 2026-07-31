---
type: concept
aliases: [大语言模型服务, LLM Inference Serving]
---

# Large Language Model Serving

## 定义

将训练完成的大语言模型通过在线 API 承载多租户 prompt prefill 与自回归 decode 请求，并在延迟、吞吐、成本和公平性之间调度资源的系统问题。

## 核心要点

1. decode 通常内存受限，因为每个 token 都需读取增长的 [[KV Cache]]。
2. batch、并行、缓存和抢占是主要优化维度。
3. 请求长度和结束时间未知，要求动态内存分配。

## 代表工作

- [[Orca]]
- [[PagedAttention]]

## 相关概念

- [[Iteration-level Scheduling]]
- [[KV Cache]]
- [[Tensor Parallelism]]
