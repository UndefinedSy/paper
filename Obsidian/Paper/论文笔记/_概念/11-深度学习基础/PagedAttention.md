---
type: concept
aliases: [分页注意力, Paged KV Attention]
---

# PagedAttention

## 定义

一种让 Transformer [[KV Cache]] 以固定大小、物理非连续的块存储，并由注意力内核依据 [[Block Table]] 逐块读取的精确注意力算法。

## 数学形式

$$
o_i=\sum_{j=1}^{\lceil i/B\rceil}V_jA_{ij}^{\top},\qquad A_{ij}\propto\exp(q_i^\top K_j/\sqrt d)$$

## 核心要点

1. 逻辑连续 KV blocks 可以映射到不连续的物理地址，不改变因果注意力的结果。
2. 只在尾块填满时申请新物理块，将每请求碎片限制在一个块内。
3. 块是共享、[[Copy-on-Write]]、交换与重计算的统一管理粒度。

## 代表工作

- [[PagedAttention]]：SOSP 2023 中提出该算法，并作为 vLLM 的 KV cache 基础。

## 相关概念

- [[KV Cache]]
- [[Block Table]]
- [[Virtual Memory]]
- [[Self-Attention]]
