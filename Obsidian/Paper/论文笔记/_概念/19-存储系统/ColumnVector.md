---
type: concept
aliases: [列向量]
---

# ColumnVector

## 定义

向量化执行引擎中承载单列批量数据的核心内存结构，是 [[VectorTableSlot]] 的组成单元。

## 核心要点

1. 对不超过 8 字节的定长数据直接存入 `vals` 数组；超长定长数据与变长数据通过 `buf` 指针间接存储，`vals` 只保存指向数据起始地址的指针。
2. `flags` 数组维护 null 状态、压缩状态等元数据，支撑后续计算的优化判断。
3. 由 [[VectorBuffer]]（`TinyBuffer` 链表）统一管理底层内存块，减少跨算子内存拷贝。

## 代表工作

- [[TDSQL]]：作为非侵入式向量化执行引擎的核心内存数据结构。

## 相关概念

- [[VectorTableSlot]]
- [[VectorBuffer]]
- [[Vectorized Execution Engine]]
