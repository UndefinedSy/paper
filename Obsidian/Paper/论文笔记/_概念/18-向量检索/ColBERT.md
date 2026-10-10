---
type: concept
aliases: [ColBERT, Contextualized Late Interaction]
---

# ColBERT

## 定义

一种基于 BERT 的高效文本检索模型，通过**上下文化晚期交互**（contextualized late interaction）在查询与文档的 token 级向量之间计算细粒度相似度，兼顾精排效果与可扩展检索效率，相比全交叉编码器大幅降低计算成本。

## 核心要点

1. **晚期交互机制**：查询与文档分别独立编码为 token 级向量集合，相似度在检索时才计算（而非像交叉编码器一样把查询与文档拼接后一起编码），兼顾表达力与可索引性。
2. **相比交叉编码器重排（如 mGTE）**：候选集较大时交叉编码器 p99 延迟可达秒级，不适合实时应用；ColBERT 式晚期交互可在毫秒级完成同等精度的重排。
3. **压缩友好**：可配合量化（Product Quantization 等）与 token-merge 操作压缩多向量存储开销，使其与稠密向量存储量级相当。

## 代表工作

- [[VikingMem]]: Memory Retrieve 模块的 Multi-vector Rerank 策略受 ColBERT 启发，预计算存储记忆的 ColBERT 向量用于高效重排，仅增加约 6.8ms p95 延迟但带来约 3.64 分（LOCOMO LLM-Judge）的效果提升。

## 相关概念

- [[mGTE]]
- [[Product Quantization]]
- [[Token-Merge]]
