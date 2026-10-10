---
type: concept
aliases: [Mirix, MIRIX Multi-Agent Memory System]
---

# Mirix

## 定义

一个多智能体记忆系统（Multi-Agent Memory System），使用六个专用记忆模块对不同类型信息分别建模，并执行跨模块的混合检索。由 Wang & Chen 提出（2025）。

## 核心要点

1. **模块化设计**：六个专用模块分别处理不同粒度/类型的记忆内容，检索时跨模块混合召回。
2. **效果**：在 VikingMem 论文的 LOCOMO 对比中是效果第二强的基线（Overall 78.66%/85.38%），验证了"专用模块能提升记忆质量"的直觉。
3. **致命缺陷——延迟**：p50 延迟高达 10.90s、p95 高达 25.76s，比 VikingMem（p50 ~0.2s）慢两个数量级，不适合交互式应用；在 LongMemEval 上甚至因超出 24 小时端到端评测时限而未能完成评测、结果未被报告。
4. 这是论文论证"复杂多模块架构≠生产可用"的关键反例。

## 代表工作

- [[VikingMem]]: 对比基线中效果第二强、但延迟开销最大的系统，用于凸显 VikingMem 在效果与延迟之间的平衡优势。

## 相关概念

- [[Memory Base]]
- [[Zep]]
