---
type: concept
aliases: [算子融合, Fused Kernel]
---

# Kernel Fusion

## 定义

把原本多个 GPU kernel 执行的连续操作合并为一个 kernel，使中间结果在寄存器或 [[SRAM]] 中直接被下游操作消费，减少 kernel launch 和 [[High Bandwidth Memory|HBM]] 的读写。

## 核心要点

1. 对 elementwise/reduction 算子，融合常能直接减少 memory traffic。
2. 训练场景仅融合不足以消除为 backward 保存的大中间张量；还需要 [[Recomputation]] 或其他存储策略。
3. FlashAttention 融合 $QK^\top$、mask、softmax、dropout 与 $PV$，并用 tiling 防止 attention tiles 外溢至 HBM。

## 代表工作

- [[FlashAttention]]：将 exact attention 完整管线组织为 IO-aware CUDA kernel。

## 相关概念

- [[Tiling]]
- [[SRAM]]
- [[Recomputation]]
