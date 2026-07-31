---
type: concept
aliases: [重计算, Activation Recomputation]
---

# Recomputation

## 定义

不持久保存某些中间激活，而在反向传播或后续阶段从较小的输入/状态重新计算它们，以降低峰值内存或数据移动的技术。

## 核心要点

1. 传统观点将其视为“以更多 FLOPs 换更少显存”。
2. 若被省去的是大而低算术强度的 [[High Bandwidth Memory|HBM]] 中间状态，重算还可能更快。
3. [[FlashAttention]] 前向保存 $O,m,\ell$，反向在 [[SRAM]] 中重算局部 $S,P$，避免存取完整 $N\times N$ attention matrix。

## 代表工作

- [[FlashAttention]]：将重算用于减少 attention backward 的 HBM IO。

## 相关概念

- [[Gradient Checkpointing]]
- [[Kernel Fusion]]
- [[Memory-Bound Operation]]
