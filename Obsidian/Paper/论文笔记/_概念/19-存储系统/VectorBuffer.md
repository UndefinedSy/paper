---
type: concept
aliases: [向量缓冲区]
---

# VectorBuffer

## 定义

统一管理向量化执行内存块的结构，实现为多个 `TinyBuffer` 节点组成的链表，每个 `TinyBuffer` 代表一段连续可用内存块。

## 核心要点

1. 通过自维护的缓冲管理机制，确保变量连续存储，减少缓存缺失（cache miss）。
2. 大变量单独存储可减少算子间的内存拷贝，加快内存复用速度。
3. 是 [[ColumnVector]] 间接存储（超长定长/变长数据）的底层内存供给方。

## 代表工作

- [[TDSQL]]：解决大规模数据存储的内存管理挑战。

## 相关概念

- [[ColumnVector]]
- [[VectorTableSlot]]
