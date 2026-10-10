---
type: concept
aliases: [Step序列, Variable-Size Allocation Scheme]
---

# Step Sequence

## 定义

一种变长空间分配方案：在传统 2 的幂次分配（power-of-2 allocation）基础上，于每个 2 的幂次区间内插入若干倍数系数档位，提供更细粒度的分配规格，从而降低内部碎片。

## 数学形式

$$
S_i = \text{block\_size} \times j \times 2^k, \quad j \in \{1, 3, 5, 7\},\ k \ge 0
$$

均匀分布数据下的期望浪费率：

$$
E[W] = \frac{1}{N-1} \sum_{i=1}^{n} \frac{(S_i - S_{i-1})^2}{2S_i}
$$

## 核心要点

1. 传统 2 的幂次分配（size class 为 $1, 2, 4, 8, 16, \dots$）每个区间跨度大，大量中间大小的对象被迫向上取整到下一档位，造成约 25%–30% 的空间浪费。
2. Step Sequence 在同一个 2 的幂次区间内额外插入 $j \in \{1,3,5,7\}$ 四个档位，细化粒度而不显著增加档位总数（管理复杂度可控）。
3. 对 64MB 量级的数据范围，该方案可将期望浪费率降至约 7.93%，是传统方案的三分之一左右。
4. 该技术是空间效率与分配管理复杂度之间权衡的典型实例，类似思路也出现在通用内存分配器（如 slab allocator）设计中。

## 代表工作

- [[SOS]]：dstorage 磁盘存储引擎用 Step Sequence 决定 block 规格集合，是实现 ≈7.93% 内部碎片率的核心机制。

## 相关概念

- [[Hash Index]]
