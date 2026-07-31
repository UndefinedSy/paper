---
type: concept
aliases: [KV 缓存重计算, Rematerialization]
---

# KV Cache Recomputation

## 定义

被抢占序列恢复时，不搬回旧 KV cache，而是把原 prompt 与已生成 token 合并为新 prefill，一次性重新计算全部历史 K/V 的恢复策略。

## 核心要点

1. 利用 prefill 的并行矩阵计算，可能远快于逐 token 原 decode。
2. 不依赖块传输，因此开销基本不随 block size 改变。
3. [[PagedAttention]] 实验中，小 block 时常优于 [[KV Cache Swapping]]。

## 代表工作

- [[PagedAttention]]：作为 sequence group 抢占后的两种恢复机制之一。

## 相关概念

- [[Preemption]]
- [[KV Cache Swapping]]
- [[KV Cache]]
