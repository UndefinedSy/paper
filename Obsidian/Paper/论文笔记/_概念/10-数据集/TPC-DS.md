---
type: concept
aliases: [TPC Decision Support Benchmark]
---

# TPC-DS

## 定义

面向大规模决策支持系统、包含复杂查询和多维数据模型的行业标准 benchmark。

## 核心要点

1. 通过 scale factor 控制数据规模。
2. 查询覆盖 join、aggregation、subquery 和复杂报表。
3. ByteHouse 使用 SF-1000 比较延迟分位数。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。
- [[TDSQL]]：以 10,000 GB (SF=10000) 配置在官方 TPC-DS 报告中取得全球第一。

## 相关概念

- [[ClickBench]]
- [[TPC-H]]
