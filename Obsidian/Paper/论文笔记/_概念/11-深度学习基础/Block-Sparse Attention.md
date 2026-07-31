---
type: concept
aliases: [块稀疏注意力, Block Sparse Attention]
---

# Block-Sparse Attention

## 定义

将 attention matrix 按行/列 tile 分块，用预定义块掩码只计算非零 blocks 的稀疏注意力机制。

## 核心要点

1. 相较 dense attention，它改变连接模式，通常是近似或任务结构假设，而不是同一 exact 语义。
2. 块粒度比任意元素稀疏更匹配 GPU tile/kernel 与内存访问。
3. 若非零块比例为 $s$，block-sparse FlashAttention 的 IO 为 $\Theta(Nd+N^2d^2s/M)$。

## 代表工作

- [[FlashAttention]]：跳过零 tiles，以 fixed butterfly mask 将 64K 序列纳入可训练范围。

## 相关概念

- [[FlashAttention]]
- [[Tiling]]
- [[Longformer]]
