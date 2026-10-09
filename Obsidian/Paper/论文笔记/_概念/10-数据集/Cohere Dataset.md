---
type: concept
aliases: [Cohere-10M, Cohere10M, Cohere]
---

# Cohere Dataset

## 定义

由 Cohere embedding 模型生成的文本嵌入数据集，是评测现代向量检索系统在大规模文本检索场景下表现的常用 benchmark。

## 核心要点

1. **规模**：常用版本 Cohere-10M 含 **1000 万条 768 维**文本嵌入向量。
2. **来源**：Cohere 官方公开的 Datasets and Embedding 资源。
3. **在评测中的角色**：代表 RAG 类文本检索负载，规模介于 GIST-1M 与内部 Tongyi-100M 之间。
4. **价值**：用于检验系统在**存储占用**与**写入吞吐**上的表现——Nova 在 Cohere-10M 上索引仅 3.0GB，而 Milvus 需 118.95GB。

## 代表工作

- [[Nova]]: 用于整体性能、索引构建时间与存储占用对比。

## 相关概念

- [[GIST Dataset]]
- [[Tongyi Dataset]]
- [[VectorDBBench]]
- [[RAG]]
