---
type: concept
aliases: [HBM, 高带宽内存]
---

# High Bandwidth Memory

## 定义

GPU 上容量大、带宽高但相对片上存储仍较慢的主显存层。GPU kernel 通常从 HBM 加载输入，使用寄存器/SRAM 计算，再将输出写回 HBM。

## 核心要点

1. 在 A100 上，HBM 大约提供 1.5–2.0 TB/s 带宽与 40–80 GB 容量。
2. HBM 带宽远低于 [[SRAM]]，因此 softmax、dropout 等低算术强度操作常是 [[Memory-Bound Operation|memory-bound]]。
3. [[FlashAttention]] 避免读写 $N\times N$ attention 中间状态，从而显著减少 HBM traffic。

## 代表工作

- [[FlashAttention]]：以 IO-aware tile schedule 优化 SRAM 与 HBM 之间的传输。

## 相关概念

- [[SRAM]]
- [[Tiling]]
- [[Kernel Fusion]]
