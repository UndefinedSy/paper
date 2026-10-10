---
type: concept
aliases: [LOCOMO, Long-term Conversational Memory Benchmark]
---

# LOCOMO

## 定义

一个高质量的长期对话记忆评测基准，用于评估大语言模型 Agent 在长期会话中的记忆能力。由 Maharana 等人在 ACL 2024 提出。

## 核心要点

1. **规模**：包含 10 组对话，每组平均 1000+ 条消息，token 量较大。
2. **标注**：每组对话配有问答标注，覆盖单跳（single-hop）、多跳（multi-hop）、时间推理（temporal reasoning）、开放域知识（open-domain）四类推理类型。
3. 是 [[VikingMem]] 等长期记忆系统论文常用的核心评测基准之一，常与 [[LongMemEval]] 搭配使用以验证不同时程规模下的记忆效果。

## 代表工作

- [[VikingMem]]: 在 GPT-4o-mini、GPT-4.1-mini 两套评测设置下均取得最高 LLM-Judge Overall 分数（88.83%、90.12%）与最佳 F1 分数（54.98%）。

## 相关概念

- [[LongMemEval]]
- [[Memory Base]]
