---
type: concept
aliases: [内存受限, Memory Bound]
---

# Memory-Bound Operation

## 定义

运行时间主要由从慢速存储读取和写入数据决定，而非由算术计算吞吐决定的操作类型。其 [[Arithmetic Intensity|算术强度]]较低。

## 核心要点

1. elementwise、reduction、softmax、dropout 和 normalization 往往属于 memory-bound。
2. 在 GPU 上，减少 [[High Bandwidth Memory|HBM]] traffic 往往比减少少量 FLOPs 更有效。
3. [[FlashAttention]] 利用 [[Tiling]]、[[Kernel Fusion]] 和 [[Recomputation]] 避免 attention 中间矩阵的 HBM 往返。

## 代表工作

- [[FlashAttention]]：展示了更多 FLOPs 但更少 IO 可获得更低 wall-clock time。

## 相关概念

- [[Arithmetic Intensity]]
- [[Kernel Fusion]]
- [[High Bandwidth Memory]]
