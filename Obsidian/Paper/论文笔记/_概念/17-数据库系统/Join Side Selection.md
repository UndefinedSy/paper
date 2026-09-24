---
type: concept
aliases: [连接侧选择, JSS]
---

# Join Side Selection

## 定义

为 hash join 等算子选择 build/probe 输入侧，以降低内存、构建和探测成本的优化决策。

## 核心要点

1. 通常倾向把更小输入置于 build side。
2. 错误 cardinality 会导致 build side 选择失误。
3. ByteHouse 用分类器自底向上局部交换 join 输入。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Cascades Optimizer]]
- [[History-Based Optimization]]
