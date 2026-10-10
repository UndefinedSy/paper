---
type: concept
aliases: [Aggregate Expression, 聚合表达式, AggregateExpression]
---

# Aggregate Expression

## 定义

[[Event-Entity Model]] 中附着在实体属性上的规则表达式，显式规定该属性如何被哪类事件、以何种算子更新。

## 核心要点

1. **结构字段**：`Op`（算子，如 LLM_MERGE/SUM）、`EventType`（参与聚合的事件类型名）、`PropertyName`（该事件类型中参与聚合的属性名）、`IsPrimaryKey`（是否为主键）。
2. 它是实体 schema 中"状态演化规则"的落地形式——定义了"哪些事件触发该属性的演化"。
3. 与算子库配合，使实体不再是静态笔记，而是从事件流**增量物化**的持久状态。

## 代表工作

- [[VikingMem]]: Entity Schema 中每个 Property 都附带一个 AggregateExpression，是其状态演化机制的核心载体。

## 相关概念

- [[Event-Entity Model]]
- [[Entity Update Algorithm]]
