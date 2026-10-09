---
type: concept
aliases: [Model Studio, 阿里云百炼, 通义模型工作室]
---

# Alibaba Cloud Model Studio

## 定义

阿里云的 LLM 运维（LLMOps）平台，覆盖模型评测、微调、部署与推理的完整生命周期，支持用户在通义（Tongyi）模型族上构建应用。

## 核心要点

1. **[[RAG]] 服务**：用户上传文档构建私有知识库，文档被切分为 chunk、生成 embedding，与元数据分别存入两张表（`t_embedding`、`t_metadata`），以 `doc_id` 关联。
2. **高密度多租户**：单实例服务 1 万–3 万用户、平均 300 并发请求，要求防止 [[Noisy Neighbor]] 效应。
3. **规模**：帮助 40 万+ 用户创建 60 万+ 数据库，管理 1000 亿+ 向量。
4. 解析与 embedding 生成在独立的、水平可扩展的层级上运行，并通过消息队列异步向数据库投递向量；该层算力远超数据库，因此**持续写入瓶颈落在向量数据库上**。

## 代表工作

- [[Nova]]: 作为 Model Studio 知识库管理的存储后端，是其 24 小时生产压测（Figure 16）的真实场景来源。

## 相关概念

- [[RAG]]
- [[AnalyticDB-PG]]
- [[Noisy Neighbor]]
