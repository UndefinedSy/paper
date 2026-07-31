---
type: concept
aliases: [张量模型并行, Tensor Model Parallelism]
---

# Tensor Parallelism

## 定义

把同一层的张量计算（如注意力头或线性层维度）切分到多张 GPU 上同步执行，并以 collective communication 聚合中间结果的模型并行方式。

## 核心要点

1. 每张 GPU 处理相同 token 集合、只保存自己的 attention heads 对应 KV 分片。
2. 控制面可广播一份共同的 [[Block Table]]，避免 worker 间的内存管理同步。
3. 计算期间通常由 all-reduce 聚合结果。

## 代表工作

- [[PagedAttention]]：支持 Megatron-LM 风格张量并行，中央 scheduler 分发块表。

## 相关概念

- [[Large Language Model Serving]]
- [[KV Cache]]
- [[Block Table]]
