---
type: concept
aliases: [Reciprocal Rank Fusion, RRF]
---

# RRF (Reciprocal Rank Fusion)

## 定义

RRF 是一种轻量级且有效的排名聚合算法，将多个排序列表融合为统一排名。

## 数学形式

$$
\text{RRF\_score}(d) = \sum_{r \in R} \frac{1}{k + r(d)}
$$

## 核心要点

1. $k$ 为常数（通常 60），$r(d)$ 为文档 $d$ 在排名列表 $r$ 中的排名
2. 不需要分数校准，适合融合不同来源的排名（向量相似度 + BM25）
3. 实现简单，效果优于 Condorcet 和单独排名学习

## 代表工作

- [[LindormVector]]: 用 RRF 融合向量搜索和全文搜索结果

## 相关概念

- [[Hybrid Search]]
