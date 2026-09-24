---
type: concept
aliases: [Estore 列存引擎]
---

# Estore

## 定义

[[TDSQL]] 内置的混合列存引擎，采用 LSM 风格架构，用行存 Stash 吸收混合写入，用列存 Silos 支撑分析型批量扫描。

## 核心要点

1. **Silos**：每列数据分片存储，每个 Silo 持有固定行数、可变物理大小的数据。
2. **Registry**：元数据目录，记录每个 Silo 的 `col_id`、`silo_id`、`silo_prt`、`row_count`、`min/max` 等统计信息，支撑系统级管理与谓词下推裁剪。
3. **Stash**：行存"暂存区"，缓冲小批量插入、流式写入及 OLTP 混合负载，避免直接写入 Silo/Registry 造成碎片化；达到阈值后周期性 merge 回 Silo。
4. 作为非侵入式向量化执行引擎的底层存储基座，兼容遗留标量执行路径。

## 代表工作

- [[TDSQL]]：提出 Estore 解决高吞吐混合写入与列存分析扫描之间的阻抗失配问题。

## 相关概念

- [[LSM-tree]]
- [[Vectorized Execution Engine]]
- [[ColumnVector]]
