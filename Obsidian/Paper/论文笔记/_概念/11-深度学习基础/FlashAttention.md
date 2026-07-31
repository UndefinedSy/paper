---
type: concept
aliases: [闪存注意力, IO-aware Attention]
---

# FlashAttention

## 定义

一种保持 [[Self-Attention|dense exact attention]] 语义的 GPU attention 算法。它将 $Q,K,V$ 按 tile 调入 [[SRAM]]，用 [[Online Softmax]] 增量合并各 tile 的归一化统计量，并避免把 $N\times N$ score/probability 矩阵写入 [[High Bandwidth Memory|HBM]]。

## 核心要点

1. 目标是降低 HBM 读写而非减少 $O(N^2d)$ 算术量。
2. 前向仅持久保存输出和每行 $(m,\ell)$；反向重算局部 attention，额外内存为 $O(N)$。
3. 在 $M$ 为 SRAM 大小时，dense HBM IO 为 $\Theta(N^2d^2/M)$；对一段 SRAM 范围渐近 IO-optimal。
4. [[Kernel Fusion]] 将 matmul、mask、softmax、dropout 与第二个 matmul 留在同一 CUDA kernel 内。

## 代表工作

- [[FlashAttention]]：Dao et al., NeurIPS 2022，提出 IO-aware exact attention。
- [[PagedAttention]]：优化 serving 中持久 [[KV Cache]] 的块式布局，与 FlashAttention 的临时 attention IO 优化互补。

## 相关概念

- [[Tiling]]
- [[Online Softmax]]
- [[Recomputation]]
- [[Block-Sparse Attention]]
- [[High Bandwidth Memory]]
- [[SRAM]]
