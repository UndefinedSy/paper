---
type: concept
aliases: [KV 缓存换出, CPU Offloading]
---

# KV Cache Swapping

## 定义

GPU KV cache pool 不足时，将被抢占序列的全部 KV blocks 复制到 CPU RAM，待有 GPU 空间时再换回继续 decode 的恢复方式。

## 核心要点

1. [[PagedAttention]] 以整个共享 sequence group 作为 all-or-nothing 换出单位。
2. CPU swap 空间可限制在 GPU KV cache pool 容量内。
3. 小 block 会产生大量小 PCIe 传输，降低有效带宽；中等块大小时与 [[KV Cache Recomputation]] 接近。

## 代表工作

- [[PagedAttention]]：将 KV block 从 GPU allocator 迁移至 CPU allocator。

## 相关概念

- [[Preemption]]
- [[KV Cache Recomputation]]
- [[Virtual Memory]]
