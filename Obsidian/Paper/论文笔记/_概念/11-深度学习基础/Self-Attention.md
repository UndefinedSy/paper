---
type: concept
aliases: [自注意力, Self Attention]
---

# Self-Attention

## 定义

Transformer 中每个 token 用自身 query 对上下文所有 key 计算权重，并对 value 加权聚合的机制。

## 数学形式

$$\operatorname{Attention}(Q,K,V)=\operatorname{softmax}(QK^\top/\sqrt d)V$$

## 核心要点

1. 自回归生成只允许读取当前位置之前的 K/V。
2. 历史 K/V 会被缓存为 [[KV Cache]]。

## 代表工作

- [[Transformer]]
- [[PagedAttention]]：以非连续物理块保存 K/V。
- [[FlashAttention]]：以 SRAM tiling 和在线 softmax 避免物化 attention matrix。

## 相关概念

- [[Causal Attention]]
- [[KV Cache]]
- [[FlashAttention]]
