---
type: concept
aliases: [算术密度]
---

# Arithmetic Intensity

## 定义

单位内存访问字节对应的算术操作次数，用于判断硬件执行中计算吞吐还是内存带宽更可能成为瓶颈。

## 核心要点

1. 高算术强度的矩阵乘法常更接近 compute-bound；softmax/dropout 等常更接近 memory-bound。
2. 提升数据复用、减少 HBM 往返，能有效提高有效算术强度。
3. [[FlashAttention]] 不通过改变 dense attention FLOP 阶数提速，而通过减少中间矩阵 IO 改善实际执行效率。

## 代表工作

- [[FlashAttention]]：以 HBM read/write 对 wall-clock time 的影响阐释 IO-aware 优化。

## 相关概念

- [[Memory-Bound Operation]]
- [[High Bandwidth Memory]]
- [[Tiling]]
