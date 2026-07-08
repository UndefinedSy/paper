---
type: concept
aliases: [LindormSearch, Lindorm搜索引擎]
---

# LindormSearch

## 定义

LindormSearch 是 Lindorm 多模型数据库的搜索引擎，提供全文和多维查询能力，可作为 LindormTable 宽表的二级索引。

## 核心要点

1. 全文检索 + 多维查询
2. 可作为 LindormTable 的二级索引加速复杂查询
3. 在混合检索中负责词法匹配（倒排索引 + [[BM25]]）
4. 结果与向量搜索结果通过 [[RRF]] 融合

## 代表工作

- [[LindormVector]]: 混合检索的词法检索组件

## 相关概念

- [[Hybrid Search]]
- [[BM25]]
- [[RRF]]
- [[LindormTable]]
