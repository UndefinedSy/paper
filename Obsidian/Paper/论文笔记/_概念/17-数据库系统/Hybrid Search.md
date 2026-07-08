---
type: concept
aliases: [Hybrid Search, 混合检索]
---

# Hybrid Search

## 定义

混合检索集成向量搜索和全文搜索，提供同时语义有意义和词法精确的检索结果。

## 核心要点

1. [[LindormVector]] 检索向量相似度候选
2. [[LindormSearch]] 检索词法匹配候选（倒排索引 + BM25）
3. 两路结果用 [[RRF]] 融合
4. Lindorm 多引擎架构天然支持

## 相关概念

- [[Filtered Search]]
- [[RRF]]
- [[LindormVector]]
