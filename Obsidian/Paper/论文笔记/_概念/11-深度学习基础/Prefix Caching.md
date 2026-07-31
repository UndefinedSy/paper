---
type: concept
aliases: [提示词缓存, Shared Prefix Cache]
---

# Prefix Caching

## 定义

缓存多个请求共用的 prompt 前缀的 [[KV Cache]]，使后续请求直接映射复用其 KV states，只计算各自不同后缀的推理优化技术。

## 核心要点

1. 特别适用于长 system prompt、few-shot demonstrations 和模板化 agent 请求。
2. 若共享块的最后一块会被后缀写入，应采用 [[Copy-on-Write]] 保证隔离。
3. 降低 prefill 计算与 KV 内存重复占用，收益随共享前缀增长而上升。

## 代表工作

- [[PagedAttention]]：将预定义共享 prefix 映射到公共物理 KV blocks；80-token/341-token prefix 分别带来 1.67×/3.58× 对 Orca(Oracle) 的吞吐。

## 相关概念

- [[KV Cache]]
- [[PagedAttention]]
- [[Copy-on-Write]]
