---
type: concept
aliases: [cost-based adaptive join, 基于代价的自适应连接, Cost-Based Join Selection]
---

# Cost-Based Adaptive Join

## 定义

在向量检索与关系连接的混合查询中，优化器根据估算代价在「先向量后连接」与「先过滤后连接」两类执行策略之间动态选择的机制。

## 数学形式

ANN 驱动 Hash Join 的代价：

$$
Cost_{HJ} \propto \min\left(k \cdot \sigma \cdot \frac{N}{N_f},\ S_{max}\right)
$$

元数据驱动 Join 的代价：

$$
Cost_{NL} \propto N_f \times R_{ppi}
$$

## 核心要点

1. **ANN-driven Hash Join**：先在 embedding 表做大规模 [[ANNS|ANN]] 扫描并建哈希表，再用过滤后的元数据表探测；代价随逆选择率 $N/N_f$ 增长，被 $S_{max}$ 截断。
2. **Metadata-driven Join**：先过滤元数据、按 `doc_id` 连接取出候选 chunk，再精确计算距离；代价由外循环文档数 $N_f$ 与每文档平均 chunk 数 $R_{ppi}$ 决定。
3. **切换点**：过滤选择性弱（$N_f$ 大）时选 Hash Join；选择性极强（$N_f$ 小）时选元数据驱动 Join。
4. 传统优化器在过滤选择率波动时会生成不稳定计划，该自适应策略正是为缓解这种抖动。

## 代表工作

- [[Nova]]: 首次系统化处理跨表混合检索，把该策略集成进生产级关系优化器。

## 相关概念

- [[Hybrid Search]]
- [[ANN-driven Hash Join]]
- [[Metadata-driven Join]]
- [[RAG]]
