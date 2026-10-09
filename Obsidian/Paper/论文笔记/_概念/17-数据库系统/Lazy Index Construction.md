---
type: concept
aliases: [lazy index construction, 惰性索引构建]
---

# Lazy Index Construction

## 定义

允许在空表上预先创建索引对象、但推迟实际索引结构构建，直到累积数据达到训练/采样阈值时才真正建库并灌入数据的策略。

## 核心要点

1. **动机**：部分向量索引需要足够的训练数据——例如聚类类方法（[[Inverted File Index|IVF]]、[[Product Quantization|PQ]]）需要足够向量才能可靠地学习质心或码本。
2. **做法**：预先创建索引对象，把构建参数持久化进元数据；基索引保持为空，delta 文件在达到采样阈值前不建索引。
3. **触发**：阈值到达后触发构建，并把 delta 中缓冲的数据灌入基索引。
4. **收益**：简化系统集成，使 DDL 与数据加载解耦，避免小数据量强行建索引造成的低质量聚类。

## 代表工作

- [[Nova]]: 在 delta-base 框架中实现，配合 [[Delta File]] 的缓冲能力。

## 相关概念

- [[Delta File]]
- [[Delta-Base Architecture]]
- [[Inverted File Index]]
