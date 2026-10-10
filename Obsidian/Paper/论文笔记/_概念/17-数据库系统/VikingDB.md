---
type: concept
aliases: [VikingDB, Viking 向量引擎]
---

# VikingDB

## 定义

字节跳动（ByteDance）旗下的云原生向量数据库引擎（亦称 BytePlus VectorDatabase），作为 [[VikingMem]] 的底层持久化与检索存储层。

## 核心要点

1. 在 [[VikingMem]] 架构中承担持久存储职责：事件记忆持久化为带文本、元数据、混合检索特征的可搜索记录；实体记忆维护为独立更新的状态记录；关键词图物化为辅助关联索引。
2. VikingMem 实现记忆专属逻辑（schema 驱动抽取、去重、算子化事件-实体更新）于 VikingDB 之上，二者分层解耦：VikingDB 管存储，VikingMem 管记忆语义。
3. 论文报告的检索延迟数字（p50/p95）包含 VikingDB 候选生成与 VikingMem 侧分数融合、重排的总耗时。

## 代表工作

- [[VikingMem]]: 构建于 VikingDB 之上的端到端 Memory Base 管理系统。

## 相关概念

- [[Memory Base]]
- [[OpenViking]]
