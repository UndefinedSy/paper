---
type: concept
aliases: [谓词下推选择, PPS]
---

# Predicate Pushdown Selection

## 定义

根据谓词选择性与计算/I-O 代价决定哪些条件值得下推，而不是无条件下推全部谓词。

## 核心要点

1. 昂贵 UDF 或向量条件可能下推后更慢。
2. ByteHouse 把 WHERE 顶层合取项编码为 AST 特征。
3. 回归模型预测 predicate-specific I/O cost。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[History-Based Optimization]]
- [[Runtime Filter]]
