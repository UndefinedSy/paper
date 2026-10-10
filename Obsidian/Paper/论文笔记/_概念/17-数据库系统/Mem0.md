---
type: concept
aliases: [Mem0, Mem0 Memory System]
---

# Mem0

## 定义

一个面向生产级 AI Agent 的可扩展长期记忆系统，从对话中抽取候选事实（facts），通过 ADD/UPDATE/DELETE/NOOP 四种操作更新记忆，并用语义搜索检索记忆。由 Chhikara 等人在 ECAI 2025 提出。

## 核心要点

1. **记忆更新机制**：对每条新抽取的事实，判断应执行 ADD（新增）、UPDATE（更新已有记忆）、DELETE（删除冲突记忆）或 NOOP（无操作）。
2. **检索方式**：基于语义搜索（向量相似度）直接检索相关记忆片段，属于相对轻量级的"插入-检索"式记忆范式。
3. **延迟优势**：检索延迟低（LOCOMO p50 0.15s、p95 0.20s），是效率最高的基线之一，但 VikingMem 论文指出其在复杂多跳/时间推理任务上表现较弱，归因于其依赖简短记忆单元难以支撑复杂推理。

## 代表工作

- [[VikingMem]]: 核心对比基线之一，在 LOCOMO/LongMemEval 两基准的 LLM-Judge 分数上均显著落后于 VikingMem。

## 相关概念

- [[Mem0-graph]]
- [[Memory Base]]
- [[Zep]]
