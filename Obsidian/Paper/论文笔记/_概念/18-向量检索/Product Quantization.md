---
type: concept
aliases: [PQ, 乘积量化]
---

# Product Quantization

## 定义

乘积量化（PQ）将 $d$ 维向量拆分为 $m$ 个不相交的子空间，在每个子空间内独立进行 K-Means 聚类，将每个子向量编码为其最近质心的索引（$b$ bits），总存储为 $m \times b$ bits。

## 数学形式

$$
\text{PQ}(\mathbf{v}) = (c_1, c_2, \ldots, c_m), \quad c_i = \arg\min_j \|\mathbf{v}_i - \boldsymbol{\mu}_{i,j}\|
$$

查询时，将查询向量同样拆分，预计算各子空间到 $2^b$ 个质心的距离，近似距离通过求和子空间距离得到。

## 核心要点

1. 极致压缩：从 $4d$ 字节（float32）压缩到 $m \times b / 8$ 字节
2. 快速距离计算：预计算查询到质心的距离表，查表求和即可
3. SIMD 加速：可用 SIMD 并行计算多个 PQ 码的距离（[[PQFastScan]]）
4. 近似误差：不可避免，需最终重排阶段弥补

## 代表工作

- [[LindormVector]]: 使用 PQ 压缩全精度向量，驻留内存实现快速扫描
- Jégou et al. (2010): PQ 原始论文

## 相关概念

- [[IVFPQ]]
- [[RaBitQ]]
- [[SIMD]]
- [[PQFastScan]]
