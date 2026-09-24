---
title: "Retrieval-Augmented Generation (RAG)"
category: "数据库系统"
tags: [llm, retrieval, vector-database]
created: 2026-08-07
---
# Retrieval-Augmented Generation (RAG)

[[RAG]] 在 LLM 生成前检索外部知识库，将相关文档或片段作为上下文，用可追溯证据降低幻觉并增强领域知识时效性。

## 在 GaussDB-Vector 中

[[GaussDBVector]] 将 embedding 持久化为关系表中的向量列，并提供实时 ANN、标量过滤、更新和跨节点扩展，适配需要持续写入知识的生产 RAG。

## 关联
- [[GaussDBVector]]
- [[Hybrid Search]]
- [[Approximate Nearest Neighbor Search]]
