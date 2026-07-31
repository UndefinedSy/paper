---
type: concept
aliases: [块表, Logical-to-Physical Block Mapping]
---

# Block Table

## 定义

为一条生成序列维护的逻辑 KV block 到物理 GPU/CPU KV block 的映射表，通常还记录尾块已填充 token 数。

## 数学形式

$$
\mathrm{BlockTable}_s[j]=p_j
$$

其中序列 $s$ 的逻辑块 $j$ 映射到物理块标识 $p_j$。

## 核心要点

1. 解耦 token 的逻辑序列顺序与物理存放位置。
2. 让 [[PagedAttention]] 内核可读取非连续的 KV cache。
3. 多条序列可在表中指向同一物理块，以引用计数表达共享。

## 代表工作

- [[PagedAttention]]：使用块表支持按需分配、共享和分布式 worker 间的一致映射。

## 相关概念

- [[PagedAttention]]
- [[KV Cache]]
- [[Virtual Memory]]
