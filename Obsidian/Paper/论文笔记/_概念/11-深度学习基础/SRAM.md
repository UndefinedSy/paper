---
type: concept
aliases: [片上静态随机存取存储器, Shared Memory]
---

# SRAM

## 定义

GPU streaming multiprocessor 内部容量很小、带宽极高的片上存储。它适合暂存可复用的计算 tile，却无法容纳完整长序列 attention matrix。

## 核心要点

1. SRAM 的容量约束决定 tile 的行/列大小和循环顺序。
2. 其带宽通常比 [[High Bandwidth Memory|HBM]] 高一个数量级，但容量小多个数量级。
3. [[FlashAttention]] 在 SRAM 中完成 score、softmax、dropout 和 value 聚合，再仅写回紧凑输出/统计量。

## 代表工作

- [[FlashAttention]]：将片上 SRAM 作为 attention 中间状态的工作集。

## 相关概念

- [[High Bandwidth Memory]]
- [[Tiling]]
- [[Kernel Fusion]]
