---
type: concept
aliases: [Event-Entity Model, 事件-实体模型, Event-Entity Primitives]
---

# Event-Entity Model

## 定义

[[Memory Base]] 的核心数据抽象，由两类基础单元构成：**Event（事件）**——离散、带时间戳的情景记录；**Entity（实体）**——持久、不断演化的状态表示；二者通过**Operator（算子）**与**AggregateExpression（聚合表达式）**相连接。

## 数学形式

事件与实体的关系可表达为一条参数化查询（而非逐场景手写 prompt）：

$$
entity := \text{SELECT } OP(event.content) \text{ FROM Events WHERE } filters(event) \text{ GROUP BY } keys(event)
$$

## 核心要点

1. **Event**：选择性抽取出的、语义连贯的、schema 约束的记忆单元，不是基于近因的原始上下文窗口。
2. **Entity**：通过显式聚合表达式和算子从事件增量物化而来的持久状态（如用户画像、Agent 工具库），而非简单压缩的状态笔记。
3. **Operator**：应用定义的函数，决定实体状态如何被关联事件更新，分为统计类（SUM/MAX/AVG/COUNT）与 LLM 类（LLM_MERGE/TIME_COMPRESS）。
4. **范式价值**：把事件存储视为通用日志、实体存储视为该日志之上的物化视图，只需改变 `keys`、`filters`、`OP` 即可服务不同下游场景（用户画像、SOP、Agent 工作记忆等），无需修改核心引擎。
5. 应用专属 prompt 只出现在流水线边缘（事件抽取、LLM 算子内部），记忆设计从"逐 prompt 工程"升级为"事件与实体之上的可复用代数"。

## 代表工作

- [[VikingMem]]: 该模型的提出者与具体系统实现，支撑其 one-pass 抽取、EUA 实体更新与多场景复用能力。

## 相关概念

- [[Memory Base]]
- [[Aggregate Expression]]
- [[Entity Update Algorithm]]
- [[One-pass Memory Extraction]]
