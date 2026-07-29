---
type: concept
aliases: [Arrow Columnar Format, Arrow 列式内存格式]
---

# Apache Arrow

## 定义

一种语言无关的列式内存数据格式，用连续 buffer、有效性位图和类型元数据实现高效分析处理与跨系统零拷贝交换。

## 核心要点

1. 同列值连续存放，利于向量化、压缩与 cache-efficient scan。
2. 通过 schema 和标准化 buffer layout 支持不同执行引擎互操作。
3. 可变长值通常由 offsets 与 data buffers 表示。
4. [[ByteGraph-Dione]] 的 ColumnsBlock 借鉴 Arrow 的按列组织方式，但额外支持 append-only 更新、指针数组与 MVCC。

## 代表工作

- [[ByteGraph-Dione]]：在图邻接表 LeafPage 中使用 Arrow-like ColumnsBlock。

## 相关概念

- [[Adaptive Dual-Format Graph Storage]]
- [[Adjacency List]]
