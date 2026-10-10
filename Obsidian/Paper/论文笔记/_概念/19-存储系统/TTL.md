---
type: concept
aliases: [TTL, Time-To-Live, 存活时间]
---

# TTL

## 定义

Time-To-Live，为数据项设定的生命周期上限，到期后数据可被安全剔除或回收，常用于缓存失效、日志清理、分层存储等场景。

## 核心要点

1. 在 [[VikingMem]] 的 `TIME_COMPRESS` 算子中，细粒度事件在被合成为高层摘要后会被分配一个 TTL；一旦摘要已保留其显著信息，过期的低层事件可安全剪除（除非后续召回重新激活该时间线）。
2. 这是一种"渐进式巩固"（lazy merge）策略的落地：近期保持高保真，老化事件先摘要、再按 TTL 过期剔除，平衡上下文丰富度与存储/检索效率。

## 代表工作

- [[VikingMem]]: `TIME_COMPRESS` 算子为细粒度事件分配 TTL 以实现长期记忆生命周期管理。

## 相关概念

- [[Memory Base]]
- [[Statefulness and Evolution]]
