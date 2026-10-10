---
type: concept
aliases: [SOP, Standard Operating Procedure, 标准操作流程]
---

# SOP

## 定义

Standard Operating Procedure，自主智能体从历史执行轨迹中提炼出的、可复用的程序化操作流程/经验规则，用于指导未来任务的执行决策。

## 核心要点

1. 与对话历史衍生的"用户画像"等声明式记忆不同，SOP 属于**程序化记忆**，需要从 Agent 的工具调用日志、执行轨迹中提取"什么工具适合什么任务""常见失败模式""可操作建议"等经验性知识。
2. VikingMem 论文指出：窄垂直领域记忆系统（如针对聊天画像设计的 schema）无法天然容纳 SOP 这类程序化、可审计的记忆需求，这正是其提出"可泛化设计范式"的动机之一。
3. 具体落地见 VikingMem 的 **Agent Tool Memory** 用例（Figure 5）：工具调用事件被持续"演化"为携带 `suitable_for`、`failure_cases`、`suggestions` 字段的实体档案，本质上就是一种细粒度 SOP。

## 代表工作

- [[VikingMem]]: Agent Memory 场景下，Entity 实例承载了工具使用的 SOP 式经验知识（Figure 5）。

## 相关概念

- [[Agent Memory]]
- [[Generalizable Design Paradigm]]
