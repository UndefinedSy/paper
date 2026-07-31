---
type: concept
aliases: [写时复制, COW]
---

# Copy-on-Write

## 定义

多个逻辑拥有者先共享同一物理数据块，直到某一拥有者需要写入时才复制该块并写入副本的内存管理技术。

## 核心要点

1. 将复制延迟到真正发生状态分叉时，避免重复保存共享前缀。
2. 需通过引用计数确定共享块是否可原地修改。
3. [[PagedAttention]] 只复制即将写入的一个 KV block，而非整条 KV cache。

## 代表工作

- [[PagedAttention]]：用于平行采样、beam search 和共享 prefix 的可变尾块。

## 相关概念

- [[Virtual Memory]]
- [[PagedAttention]]
- [[Prefix Caching]]
