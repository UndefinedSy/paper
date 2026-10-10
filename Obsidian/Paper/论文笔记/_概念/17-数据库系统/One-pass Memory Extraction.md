---
type: concept
aliases: [One-pass Memory Extraction, 单次抽取, 单次记忆抽取范式]
---

# One-pass Memory Extraction

## 定义

VikingMem 提出的记忆抽取范式：把可配置的事件与实体 schema 编译为单一综合 prompt，LLM 对原始输入只处理一次，在同一次推理中同时抽取所有定义的记忆类型（事件 + 实体更新内容）。

## 核心要点

1. **对比传统多次抽取**：传统方法为每种记忆类型用独立、硬编码 prompt，抽取 $k$ 种类型需调用 LLM $k$ 次，造成高额 token 消耗。
2. **实现基础**：依赖 schema 的程序化编译以及 LLM 的 [[In-Context Learning|ICL]] 能力。
3. **配套优化**：使用[[Prefix Caching|前缀缓存]]复用固定 prompt 前缀（System Instruction + Memory Schema）的缓存表示，进一步降低 token 消耗。
4. **效果**（Table 2，LOCOMO，三种记忆类型）：抽取成本从 \$0.35（Multiple Prompts）降到 \$0.11（One-pass w/o EUA），即 3.2× 下降，分数几乎不变（88.86 → 88.83）。
5. 若抽取 $k$ 种记忆类型，相比多次抽取基线可节省约 $(k-1)\times$ 的 token 消耗。

## 代表工作

- [[VikingMem]]: 该范式的提出者，与 [[Entity Update Algorithm|EUA]] 组合实现"单次抽取 + 无 LLM 更新"的双重效率优化。

## 相关概念

- [[Event-Entity Model]]
- [[Entity Update Algorithm]]
