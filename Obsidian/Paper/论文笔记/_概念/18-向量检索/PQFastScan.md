---
type: concept
aliases: [PQFastScan]
---

# PQFastScan

## 定义

PQFastScan 利用 SIMD（单指令多数据）操作高效并行计算查询向量与一批 PQ 码之间的距离，显著降低单查询延迟。

## 核心要点

1. 将 PQ 码重新组织为 SIMD 友好的布局
2. 一次 SIMD 操作可处理 32 个 PQ 码的距离计算
3. 是 IVFPQ 码扫描阶段的核心加速技术

## 代表工作

- [[LindormVector]]: 使用 SIMD 加速量化向量扫描

## 相关概念

- [[Product Quantization]]
- [[IVFPQ]]
- [[SIMD]]
