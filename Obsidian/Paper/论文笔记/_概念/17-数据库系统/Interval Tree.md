---
type: concept
aliases: [Interval Tree, 区间树]
---

# Interval Tree

## 定义

一种专门支持「查询与给定点或区间有重叠的所有区间」的数据结构，通常以 BST（如 AVL 树）为基础，节点按区间端点排序，并用额外的增广信息加速重叠查询。

## 数学形式

设节点 $v$ 对应区间 $[l_v, r_v]$，增广值 $\text{max\_high}(v)$ 为该节点子树中所有区间右端点的最大值：

$$
\text{max\_high}(v) = \max\bigl(r_v,\ \text{max\_high}(\text{left}(v)),\ \text{max\_high}(\text{right}(v))\bigr)
$$

重叠搜索可借此在 $O(\log n + k)$ 时间内完成（$k$ 为命中区间数）。

## 核心要点

1. **增广 BST**：基础结构是 AVL 或红黑树，节点按区间下界排序；每个节点额外记录子树内的最大上界，用于剪枝：若当前子树的 max_high 小于查询点，则整棵子树不可能命中，直接跳过。
2. **区间节点携带负载**：在 Firestore 的 [[ChangeStream Filter]] 中，每个区间节点挂着「命中该区间的查询 ID 集合」，找到命中区间后直接取出对应查询 ID。
3. **多维扩展**：Firestore 把多字段等值条件折叠成多维元组构成的区间（如 `("Carlos", 2000)` 至 `("Carlos", ∞)`），通过元组比较复用一维区间树。
4. **静态 vs. 动态**：动态区间树支持在线插入/删除（对应查询的订阅/取消），Firestore 的实时查询正是需要动态版本。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)——[[ChangeStream Filter]] 的核心索引结构

## 相关概念

- [[ChangeStream Filter]]
- [[Trie]]
- [[QueryMatcher]]
