---
type: concept
aliases: [Agent Memory, 智能体记忆, 自主智能体记忆]
---

# Agent Memory

## 定义

面向自主智能体（autonomous agent）的记忆能力，用于捕获执行轨迹与工具使用日志，并逐步精炼为可复用的工具使用经验与标准操作流程（[[SOP]]），以提升未来任务的可靠性与效率。

## 核心要点

1. 与社交陪伴/教育等以用户画像为核心的记忆场景不同，Agent Memory 核心关注**过程性、可审计的执行知识**，而非单纯的用户偏好。
2. VikingMem 的典型落地案例（Figure 5）：从 Agent 平台原始工具调用日志中选择性抽取离散**事件**（工具名、使用情境、任务目标、成功状态等），再演化为该工具的持久**实体**档案（成功率、平均耗时、适用场景、失败案例、改进建议）。
3. 是 VikingMem 五大工业应用场景之一（社交陪伴、搜索推荐、效率协作、教育、Agent Memory）。
4. 在算子使用频率统计（Table 5）中，Agent Memory 场景高频使用 MAX（96%）、AVG（98%）、LLM_MERGE（98%），体现其对统计聚合与语义合并经验的双重依赖。

## 代表工作

- [[VikingMem]]: 典型落地场景，详见 Agent Tool Memory 用例（§4.2，Figure 5）。

## 相关概念

- [[Event-Entity Model]]
- [[SOP]]
- [[Memory Base]]
