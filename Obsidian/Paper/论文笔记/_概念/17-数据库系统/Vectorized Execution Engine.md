---
type: concept
aliases: [向量化执行引擎]
---

# Vectorized Execution Engine

## 定义

将传统逐行（tuple-at-a-time）处理转换为按列批量（column-vector batch）处理的查询执行范式，结合 SIMD 指令加速计算密集型算子。

## 核心要点

1. 计划树自底向上贪心遍历重写：子树若支持向量化则整体向量化执行，否则插入 row-vector 转换适配器兼容遗留标量算子。
2. 内存组织依赖 [[VectorTableSlot]]、[[ColumnVector]]、[[VectorBuffer]] 三层结构支撑批处理与减少内存拷贝。
3. 关键算子（[[HashAgg]]、[[HashJoin]]、RemoteSubplan、Scan）均重构为向量化实现，利用 SelectionVector/bitmap 实现分支削减。
4. 标量函数通过 metadata-driven 二分查找映射到向量化实现，避免重写数千个既有函数。

## 代表工作

- [[TDSQL]]：在成熟 MPP 生态上非侵入式集成向量化执行引擎与列存 [[Estore]]。
- [[DuckDB]]：原生向量化单机执行引擎的代表。

## 相关概念

- [[Estore]]
- [[ColumnVector]]
- [[HashAgg]]
- [[HashJoin]]
- [[SIMD]]
