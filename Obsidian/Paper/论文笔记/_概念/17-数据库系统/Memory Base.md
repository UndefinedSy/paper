---
type: concept
aliases: [Memory Base, 记忆库, 记忆基座]
---

# Memory Base

## 定义

一种面向 LLM 应用长期有状态交互的数据管理新范式，用于管理持久状态的生命周期，取代"把越来越长历史塞进上下文"或"碎片化日志/缓存"的旧做法。

## 核心要点

1. **选择性抽取（Selective Extraction）**：从原始信息流中选择性提取高价值记忆，而非全量存储或粗暴分块。
2. **内在状态性与演化（Statefulness and Evolution）**：记忆内容被渐进式摘要、修正、按时间加权，优先呈现近期交互，模拟人类记忆巩固过程。
3. **可泛化抽象范式（Generalizable Design Paradigm）**：抽象出可跨场景（教育、推荐、Agent 记忆等）迁移的记忆结构，而非为每个场景单独设计。
4. 与传统 RAG 的区别：RAG 是"检索增强生成"的信息获取手段，Memory Base 是对**持久状态本身**的生命周期管理（巩固、遗忘、溯源）。

## 代表工作

- [[VikingMem]]: 首次正式定义 Memory Base 范式，并通过 Event-Entity 模型将其具体化为可落地的管理系统。

## 相关概念

- [[Event-Entity Model]]
- [[Selective Extraction]]
- [[Statefulness and Evolution]]
- [[Generalizable Design Paradigm]]
- [[RAG]]
