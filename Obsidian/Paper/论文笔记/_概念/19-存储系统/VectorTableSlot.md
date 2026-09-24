---
type: concept
aliases: [向量表槽]
---

# VectorTableSlot

## 定义

向量化执行引擎中表示"元组向量"（tuple vector）的容器结构，取代传统行式执行中的 TupleTableSlot。

## 核心要点

1. 持有多个 [[ColumnVector]] 实例，每个对应一列的批量数据。
2. 同时维护批次相关的元数据（行数、列数）与系统向量信息，支撑批处理粒度的算子间数据传递。
3. 相比 TupleTableSlot 逐行传递，VectorTableSlot 按列批量传递，是实现向量化批处理的基础容器。

## 代表工作

- [[TDSQL]]：非侵入式向量化执行引擎的内存组织核心。

## 相关概念

- [[ColumnVector]]
- [[VectorBuffer]]
- [[Vectorized Execution Engine]]
