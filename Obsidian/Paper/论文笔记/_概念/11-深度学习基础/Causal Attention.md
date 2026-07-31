---
type: concept
aliases: [因果注意力, Masked Self-Attention]
---

# Causal Attention

## 定义

在位置 $i$ 仅允许 query 访问位置 $1\ldots i$ 的 key/value 的 self-attention 机制，用于保持自回归模型不窥视未来 token。

## 数学形式

$$a_{ij}=\frac{\exp(q_i^\top k_j/\sqrt d)}{\sum_{t=1}^{i}\exp(q_i^\top k_t/\sqrt d)},\quad o_i=\sum_{j=1}^{i}a_{ij}v_j$$

## 核心要点

1. 掩码确保 $j>i$ 的注意力权重为零。
2. 历史 K/V 是 decode 的持久状态，构成 [[KV Cache]]。
3. [[PagedAttention]] 按块重排物理地址但精确保留该计算。

## 代表工作

- [[Transformer]]
- [[PagedAttention]]

## 相关概念

- [[Self-Attention]]
- [[Autoregressive Generation]]
