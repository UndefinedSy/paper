---
type: concept
aliases: [LongMemEval, LongMemEval_s]
---

# LongMemEval

## 定义

一个更长时程的交互式记忆评测基准，用于评测聊天助手的长期记忆能力，由 Wu 等人在 ICLR 2025 提出。

## 核心要点

1. **规模**：常用子集 LongMemEval_s 包含 500 组长对话，平均约 115,000 tokens，比 [[LOCOMO]] 长约 346×。
2. **特点**：对话历史带时间戳，问题覆盖多会话推理（multi-session）、记忆更新（knowledge-update）等多种能力维度（SSU/MS/SSP/TR/KU/SSA 六类）。
3. 由于其长度远超 LOCOMO，是检验记忆系统在**超长时程、强存储压力**下效果与效率（尤其检索延迟）的关键基准。

## 代表工作

- [[VikingMem]]: 在 GPT-4o-mini、GPT-4o 两套设置下取得最高 Overall LLM-Judge 分数（66.36%、75.80%），同时保持 p95 延迟 ≤0.89s；对比基线 Mirix 甚至因 24 小时超时未能完成该基准评测。

## 相关概念

- [[LOCOMO]]
- [[Memory Base]]
