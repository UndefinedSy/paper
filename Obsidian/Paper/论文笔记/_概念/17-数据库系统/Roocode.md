---
type: concept
aliases: [Roocode, Roo-Code]
---

# Roocode

## 定义

一个开源代码编辑/生成工具，采用 `«« SEARCH ... ==== ... »» REPLACE` 格式的 patch 来对已有文本做精确局部替换。

## 核心要点

1. VikingMem 的 [[Entity Update Algorithm|EUA]] 算法直接借用了这一 patch 格式思想：抽取器输出 SEARCH/REPLACE 形式的字段级 patch，再用近似匹配定位并替换，实现无需额外 LLM 调用的确定性文本更新。
2. 该思路的核心价值在于：把"用 LLM 生成完整新文本"的开销，转化为"用 LLM 只生成差异 patch + 确定性算法应用 patch"的更低成本流程。

## 代表工作

- [[VikingMem]]: EUA 算法的设计灵感来源。

## 相关概念

- [[Entity Update Algorithm]]
- [[Edit-Distance-based Approximate Search]]
