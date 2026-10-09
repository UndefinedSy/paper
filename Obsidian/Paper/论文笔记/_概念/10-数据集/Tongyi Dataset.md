---
type: concept
aliases: [Tongyi-10M, Tongyi-100M, Tongyi]
---

# Tongyi Dataset

## 定义

阿里内部的 RAG 数据集，由文档切分后的 chunk 向量构成，包含 `t_embedding` 与 `t_metadata` 两张以 `doc_id` 关联的表，复刻 Model Studio 知识库的结构。

## 核心要点

1. **版本**：Tongyi-10M 含 1000 万条、Tongyi-100M 含 1 亿条 **1024 维**向量。
2. **结构特点**：**天然跨表**——chunk 向量与文档元数据分离存储，是 [[Nova]] 跨表混合检索实验的主要载体。
3. **用途**：用于整体性能评测与多租户大规模测试（从 Tongyi-100M 随机采样生成 300 张表，使索引总量为可用内存的 6–8 倍）。

## 代表工作

- [[Nova]]: 内部数据集，用于验证跨表混合检索与高密度多租户场景。

## 相关概念

- [[GIST Dataset]]
- [[Cohere Dataset]]
- [[RAG]]
- [[Cost-Based Adaptive Join]]
