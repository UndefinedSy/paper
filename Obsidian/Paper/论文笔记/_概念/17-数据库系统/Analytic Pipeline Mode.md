---
type: concept
aliases: [分析流水模式, APM]
---

# Analytic Pipeline Mode

## 定义

ByteHouse 面向低延迟交互分析的向量化、流水并行执行模式。

## 核心要点

1. 根据早期采样自适应选择 partial aggregation 或直接 shuffle。
2. 用 runtime filter 提前裁剪 probe 端。
3. credit-based flow control 控制背压与缓冲占用。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Runtime Filter]]
- [[Apache Arrow]]
