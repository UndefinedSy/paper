---
type: concept
aliases: [Mem0-graph, Mem0 Graph Integration]
---

# Mem0-graph

## 定义

[[Mem0]] 的知识图谱扩展版本，在抽取出的实体与关系之上构建知识图谱，支持基于图的检索。

## 核心要点

1. 相比原版 Mem0，引入图结构以捕捉实体间关系，理论上能支撑更复杂的多跳推理。
2. **代价**：动态知识图谱构建带来显著的构建时延迟，VikingMem 论文指出这类基于 KG 的记忆系统（Mem0-graph、Zep 等）在高吞吐写入场景下（单租户日均 10 亿+ token）会面临不可用的性能瓶颈。
3. 在 LOCOMO/LongMemEval 实测中检索延迟（p50 0.48s/p95 0.66s，LongMemEval 更高）明显高于普通 Mem0，印证了图维护的额外开销。

## 代表工作

- [[VikingMem]]: 对比基线之一，用于验证"知识图谱式记忆"相对"事件-实体模型"在延迟与效果上的差距。

## 相关概念

- [[Mem0]]
- [[Zep]]
