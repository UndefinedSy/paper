---
type: concept
aliases: [链式前缀哈希, Prefix-Hash Chain, Recursive Chunk Hashing]
---

# Chunk-based Prefix-Hash Chain

## 定义

一种前缀感知的分块哈希方案：将 token 序列切成多个 chunk，每个 chunk 的哈希由"自身 token id 拼接前一个 chunk 的哈希"递归计算得到，从而使两个 chunk 共享同一前缀 key 的充要条件是其所有前驱 token 完全一致。

## 核心要点

1. 保证前缀匹配的语义正确性：只要历史不同，哈希链就会分叉，不会出现伪命中。
2. 支持细粒度缓存管理：按 chunk（而非整条序列）索引和复用 KV cache，能够高效复用跨请求重叠的前缀片段。
3. 是现有多个 LLM caching 系统（如 [[LMCache]]、[[PagedAttention]]）通用的前缀缓存索引实践之一。

## 代表工作

- [[PolarKV]]：客户端库用该方案把 token 序列映射为 hash key，用于在远端内存/存储中定位 KV 条目，支撑细粒度的 [[Prefix Caching|前缀复用]]。

## 相关概念

- [[Prefix Caching]]
- [[KV Cache]]
