---
type: concept
aliases: [Edit-Distance-based Approximate Search, 基于编辑距离的近似搜索, Top-k String Similarity Search]
---

# Edit-Distance-based Approximate Search

## 定义

在字符串中以最小编辑距离为准则定位最佳匹配子串（span）的近似搜索技术，常用于容忍轻微文本差异的字符串替换/匹配场景。

## 核心要点

1. 相比精确字符串匹配，能容忍 LLM 输出的轻微不精确（如空格、措辞差异），仍能定位到语义正确的目标片段。
2. 在 [[Entity Update Algorithm|EUA]] 中用于在旧实体字段里定位 patch 的 SEARCH 片段应该替换的位置。

## 代表工作

- [[VikingMem]]: EUA 算法用其定位 patch 的最佳匹配跨度，实现无 LLM 的实体字段更新。

## 相关概念

- [[Entity Update Algorithm]]
