---
type: concept
aliases: [Metadata-driven Join]
---

# Metadata-driven Join

## 定义

跨表混合检索中的一种执行策略：先应用元数据过滤，把过滤后的元数据表与向量表按 `doc_id` 连接取出候选 chunk，再对这些 chunk 精确计算向量距离产出 top-$k$。

## 数学形式

$$
Cost_{NL} \propto N_f \times R_{ppi}
$$

## 核心要点

1. 代价由**内表访问总次数**决定：$N_f$（外循环合格文档数）乘以 $R_{ppi}$（每文档平均 chunk 数）。
2. 关系连接步骤在 Nested Loop、[[HashJoin|Hash Join]]、Merge Join 等物理算子中选择；论文重点分析 Nested Loop 变体。
3. 适合元数据过滤**高度选择性**（$N_f$ 小）的场景——外循环迭代次数少，内层扫描更经济。
4. 与 [[ANN-driven Hash Join]] 互补，切换点由 [[Cost-Based Adaptive Join]] 的代价比较决定。

## 代表工作

- [[Nova]]: Sec. 5.2 提出该策略并纳入优化器的自适应选择。

## 相关概念

- [[Cost-Based Adaptive Join]]
- [[ANN-driven Hash Join]]
- [[HashJoin]]
