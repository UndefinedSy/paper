---
type: concept
aliases: [Zep, Zep Memory Architecture]
---

# Zep

## 定义

一种时序知识图谱架构的 Agent 记忆系统，构建情景（episodic）、语义（semantic）、社区（community）三层记忆，结合时间信号做混合检索与重排。由 Rasmussen 等人提出（2025）。

## 核心要点

1. **分层记忆设计**：episodic 层保存原始交互片段，semantic 层沉淀抽象知识，community 层聚合跨会话的群体性模式。
2. **检索方式**：混合检索（结合语义相似度与时间信号）叠加重排。
3. **效果定位**：在 VikingMem 论文的对比中，Zep 是除 VikingMem、Mirix 外效果较强的基线之一（LongMemEval 上部分类别甚至领先 VikingMem），尤其在 LongMemEval 的 SSU/KU 等类别上表现突出，但 Overall 分数仍落后于 VikingMem，且检索延迟（p50 3.8s/p95 5.45s on LongMemEval）明显更高。
4. 同样属于知识图谱式记忆架构，继承了动态 KG 维护的构建延迟代价。

## 代表工作

- [[VikingMem]]: 核心对比基线之一，论文多处评测协议直接沿用 Zep 的设置以保证公平比较。

## 相关概念

- [[Mem0-graph]]
- [[Memory Base]]
