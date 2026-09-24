---
type: concept
aliases: [Trie, Prefix Tree, 前缀树, 字典树]
---

# Trie

## 定义

一种利用字符串（或有序序列）的公共前缀来组织键值对的树形数据结构，每条从根到节点的路径对应一个键的前缀，根到叶节点（或某些内部节点）的完整路径代表一个完整键。

## 数学形式

设字母表为 $\Sigma$，深度为 $d$ 的节点 $v$ 对应前缀 $\sigma_1\sigma_2\cdots\sigma_d \in \Sigma^d$；节点 $v$ 的孩子集合为 $\{(c, v') \mid \sigma \cdot c \text{ 仍是某键的前缀}\}$。

查找操作的时间复杂度为 $O(|k|)$（$|k|$ 为键长），与存储键的数量无关。

## 核心要点

1. **前缀共享**：拥有相同前缀的键共享路径节点，内存利用率远高于将键全量存储在哈希表中。
2. **有序遍历**：按字典序遍历整棵 trie 即可得到有序键集，天然支持前缀范围查询。
3. **在 Firestore ChangeStream Filter 中的应用**：字段列表作为「键」（每个字段是 $\Sigma$ 中的元素），根到节点的路径代表字段组合；路径末端挂着 [[Interval Tree]]，节点间共享字段前缀，大幅减少冗余存储。
4. **动态操作**：插入/删除单个键的代价为 $O(|k|)$，非常适合实时查询订阅/取消这类频繁变更的场景。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)——[[ChangeStream Filter]] 的外层索引结构

## 相关概念

- [[ChangeStream Filter]]
- [[Interval Tree]]
- [[QueryMatcher]]
