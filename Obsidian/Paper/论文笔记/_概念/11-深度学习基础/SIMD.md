---
type: concept
aliases: [SIMD, Single Instruction Multiple Data, 单指令多数据]
---

# SIMD

## 定义

SIMD（单指令多数据）是一种并行计算范式，一条指令同时对多个数据元素执行相同操作，利用 CPU 向量寄存器提升吞吐量。

## 核心要点

1. IVFPQ 码扫描的核心加速技术
2. 固定大小批次处理（如 32 个 PQ 码并行计算）
3. [[PQFastScan]] 重新组织 PQ 码为 SIMD 友好布局
4. LindormVector 使用 SIMD 加速量化向量距离计算

## 代表工作

- [[LindormVector]]: SIMD 加速 PQ 码扫描
- [[TDSQL]]: 加速向量化 [[HashAgg]]/[[HashJoin]] 的哈希与键比较，以及标量函数的向量化实现

## 相关概念

- [[Product Quantization]]
- [[PQFastScan]]
- [[IVFPQ]]
