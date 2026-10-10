---
type: concept
aliases: [Generalizable Design Paradigm, 可泛化设计范式]
---

# Generalizable Design Paradigm

## 定义

[[Memory Base]] 的第三条核心原则：实现一个灵活、可迁移的记忆 schema，把底层存储逻辑与具体业务规则解耦，使单一基础设施能以最小重配置代价服务差异巨大的下游场景（情感陪伴、自主智能体、教育、推荐等）。

## 核心要点

1. **动机**：不同下游应用需要截然不同的记忆结构与内容（如情感陪伴聚焦用户个性/对话风格，自主智能体需要可执行经验与 SOP），为每个场景单独建设造成高维护成本、无法共享能力。
2. **实现**：通过 JSON-like 的 [[Event-Entity Model|Event-Entity schema]] 定义记忆结构，而非把任务逻辑硬编码进 prompt。
3. 是 [[One-pass Memory Extraction|单次抽取效率]]的关键使能器——schema 驱动的设计使单次推理同时抽取多种记忆类型成为可能。

## 代表工作

- [[VikingMem]]: 该原则的具体化实现，支撑其在社交陪伴、搜索推荐、效率协作、教育、Agent 记忆五大场景的落地（§4）。

## 相关概念

- [[Memory Base]]
- [[Event-Entity Model]]
