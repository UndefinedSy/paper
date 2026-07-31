---
type: concept
aliases: [分块计算, Blocking]
---

# Tiling

## 定义

把大矩阵或张量按能装入快速存储的 tile 切分，逐块加载、计算与聚合，从而提高数据复用并减少对慢速存储的访问。

## 核心要点

1. tile 大小必须同时满足输入块、输出块和中间块的 SRAM 容量约束。
2. 正确的循环顺序决定何种数据被复用；FlashAttention 以 $K,V$ 为外层块，以减少它们的 HBM 重载。
3. 分块本身不保证数值正确；跨块 reduction 需要可组合状态，例如 [[Online Softmax]] 的最大值与归一化和。

## 代表工作

- [[FlashAttention]]：以 SRAM tiling 避免物化 attention matrix。

## 相关概念

- [[SRAM]]
- [[High Bandwidth Memory]]
- [[Kernel Fusion]]
- [[Online Softmax]]
