---
type: concept
aliases: [自适应双格式图存储, Adjacency-List-Level Format Conversion]
---

# Adaptive Dual-Format Graph Storage

## 定义

在单条 [[Adjacency List|邻接表]] 粒度，根据在线负载成本在行式 RowBlock 与列式 ColumnsBlock 之间无停机切换的图存储机制。

## 核心要点

1. 行存优化完整记录点读，列存优化少列大范围扫描。
2. 两种格式共享 Pointer Array、版本链、commit/delete timestamp 和 [[MVCC]] 接口。
3. 转换经历增量追加、快照、格式构建、增量回放和追平双写五阶段。
4. 以 [[Row-Column Cost Model]] 选择格式，并用转换阈值避免抖动。
5. 相对永久 dual format 只损失 2.2%–5.3% 吞吐，却节省 41.25%–47.25% 空间。

## 代表工作

- [[ByteGraph-Dione]]：在 append-only [[BW-Tree]] LeafPage 中实现该机制。

## 相关概念

- [[Row-Column Cost Model]]
- [[Apache Arrow]]
- [[BW-Tree]]
- [[HTAP]]
