---
type: concept
aliases: [Statefulness and Evolution, 内在状态性与演化]
---

# Statefulness and Evolution

## 定义

[[Memory Base]] 的第二条核心原则：记忆不是静态归档，而是一个生命周期——离散事件驱动持久实体的持续精炼，模拟人类记忆巩固过程，兼顾情景历史（事件）与演化状态（实体）的一致与新鲜。

## 核心要点

1. **动机**：传统"插入-检索"式静态向量存储只会累积历史、不更新底层状态，导致矛盾与过期信息；真实世界实体（学习中的用户、执行工作流的 Agent）持续变化。
2. **实现**：通过算子库（LLM_MERGE、SUM、TIME_COMPRESS 等）把事件程序化地链接到实体，驱动高效更新与渐进式巩固（策略性摘要或遗忘过期信息）。
3. **检索侧配套**：动态加权机制（时间加权 + 业务权重）在检索时优先呈现最相关上下文。
4. 对应系统实现中的 `TIME_COMPRESS` 算子（渐进式巩固，懒合并策略）与 Entity Memory 更新机制。

## 代表工作

- [[VikingMem]]: 该原则的具体化实现，通过 Operator 库与 TIME_COMPRESS 算子落地。

## 相关概念

- [[Memory Base]]
- [[Event-Entity Model]]
