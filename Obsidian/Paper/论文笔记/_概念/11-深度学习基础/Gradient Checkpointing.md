---
type: concept
aliases: [梯度检查点, Activation Checkpointing]
---

# Gradient Checkpointing

## 定义

训练时只保存部分激活作为 checkpoint，并在 backward 时重新执行若干前向计算来恢复未保存激活，以降低峰值显存的技术。

## 核心要点

1. 通常以额外计算时间交换显存容量。
2. [[FlashAttention]] 属于选择性重算：保存每行 softmax 统计量，重算局部 attention tile，目标不仅是节省显存，更是减少 [[High Bandwidth Memory|HBM]] IO。
3. 它与一般 checkpoint 的不同在于其 tile schedule、精确归一化状态和融合 kernel 都针对 attention 专门设计。

## 代表工作

- [[FlashAttention]]：以重算 $S,P$ 避免保存二次 attention 中间状态。

## 相关概念

- [[Recomputation]]
- [[Online Softmax]]
- [[Memory-Bound Operation]]
