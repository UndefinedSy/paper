---
type: concept
aliases: [Selective Extraction, 选择性信息抽取]
---

# Selective Extraction

## 定义

[[Memory Base]] 的第一条核心原则：从低密度原始信息流（如会议记录、调试日志）中，用细粒度事件抽取与分段技术剔除无关噪声（如寒暄、填充语），保留高价值离散信号，而非盲目截断或摘要整段文本。

## 核心要点

1. **动机**：传统 RAG 处理高密度、预处理过的文档，但生产记忆系统面对的是低密度、话题交织的原始流；盲目截断/摘要会导致 "context pollution"（LLM 被无关噪声分散注意力或 "over-squashed"）。
2. **要求**：需精确识别语义片段的起止位置，把分散的信息重组为连贯的"事件"。
3. **效率约束**：抽取还必须计算高效——避免多遍抽取多种记忆类型带来的 token 爆炸（见 [[One-pass Memory Extraction]]）。
4. VikingMem 的具体实现是**两阶段智能记忆分段策略**：语义显著性过滤 + 事件中心划分。

## 代表工作

- [[VikingMem]]: 原则的提出与具体化，落地为"智能记忆分段策略"（Intelligent Memory Segmentation）。

## 相关概念

- [[Memory Base]]
- [[Event-Entity Model]]
