---
type: concept
aliases: [Entity Update Algorithm, EUA, 无 LLM 实体更新算法]
---

# Entity Update Algorithm

## 定义

VikingMem 提出的 **EUA（Entity Update Algorithm）**，一种无需额外 LLM 调用即可更新实体字符串字段的确定性 patch 应用算法，受 [[Roocode]] 启发。

## 数学形式

```
Input: Entity schema S; old entity E_old; field-wise patches {p_f}
Output: Updated entity E_new
1  E_new ← E_old
2  foreach field f in S do
3      (s, r) ← ParsePatch(p_f)              // s=SEARCH, r=REPLACE
4      if s = ∅ then continue
5      (i, j) ← BestApproxSpan(E_old[f], s)   // min edit distance
6      E_new[f] ← E_old[f][0:i] ‖ r ‖ E_old[f][j:]
7  return E_new
```

## 核心要点

1. **动机**：实体字段为 string 类型时，传统做法需把抽取事件连同旧实体再喂给 LLM 合成新实体——这是第二次 LLM 调用，在延迟敏感场景（实时对话内摘要）下代价过高。
2. **做法**：抽取器直接输出紧凑 patch `«« SEARCH ... ==== ... »» REPLACE`；EUA 用[[Edit-Distance-based Approximate Search|基于编辑距离的近似搜索]]在旧字段中定位最佳匹配跨度，再替换为 REPLACE 内容。
3. **鲁棒性**：近似匹配对"轻微 LLM 不精确导致精确字符串匹配失败"具有鲁棒性。
4. **候选集限制**：每次更新先用 ANN 搜索检索一个小候选集（部署中 top-5 相关现有实体），只把这些候选纳入 patch 生成 prompt，使 prompt 长度受限但保留足够上下文。
5. **效果**（Table 2，LOCOMO）：相比"One-pass w/o EUA"，EUA 把端到端耗时从 12.32s 降到 7.42s、成本从 \$0.11 降到 \$0.07，质量几乎不变（88.83 → 88.77）。

## 代表工作

- [[VikingMem]]: EUA 的提出者，是其"单次抽取 + 无 LLM 实体更新"组合优化的关键一环。

## 相关概念

- [[Event-Entity Model]]
- [[Edit-Distance-based Approximate Search]]
- [[One-pass Memory Extraction]]
- [[Roocode]]
