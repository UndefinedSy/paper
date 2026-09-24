---
type: concept
aliases: [Vector Database Benchmark]
---

# VDBBench

## 定义

用于统一评估向量数据库吞吐、延迟与 recall 的 benchmark 框架。

## 核心要点

1. 通常固定 recall 比较 QPS。
2. 可组合 scalar filter 与向量检索。
3. ByteHouse 以 99% recall、1% scalar filter 比较 Milvus 和 pgvector。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Milvus]]
- [[pgvector]]
- [[Hybrid Search]]
