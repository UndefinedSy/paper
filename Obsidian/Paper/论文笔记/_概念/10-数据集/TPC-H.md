---
type: concept
aliases: [TPC-H Benchmark]
---

# TPC-H

## 定义

面向决策支持系统的标准关系型 benchmark，由 8 张表和 22 条分析查询构成。

## 核心要点

1. 查询包含 join、aggregation 和排序。
2. scale factor 控制数据量。
3. ByteHouse 用 SF-100 的 Q12/Q14/Q19 评估 IPM。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[TPC-DS]]
- [[Incremental Processing Mode]]
