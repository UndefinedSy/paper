---
type: concept
aliases: [Change Stream Filter, 变更流过滤器]
---

# ChangeStream Filter

## 定义

[[QueryMatcher]] 内部用于把文档变更高效路由到匹配查询的内存数据结构，由「字段列表 → [[Interval Tree]]」映射组织为 [[Trie]]，把单次变更的匹配代价从 $O(\text{queries})$ 降到 $O(\text{matches}) + O(\log \text{queries})$。

## 数学形式

查询 `where sender = "Carlos" and time > 2000` 在 `["sender", "time"]` 条目下表示为区间：

$$
\big((\text{"Carlos"},\, 2000),\ (\text{"Carlos"},\, \infty)\big)
$$

对文档变更的 pre/post-image 各求一次匹配集合 $M(\cdot)$，再取差集：

$$
\text{新命中} = M(\text{post}) \setminus M(\text{pre}),\quad \text{不再命中} = M(\text{pre}) \setminus M(\text{post})
$$

## 核心要点

1. **Trie 组织字段前缀**：每个节点持一个字段，根到节点的路径即字段列表；相同前缀的字段组合复用路径，节省内存。
2. **区间树存储查询区间**：每个 Trie 路径端点挂一棵 AVL 区间树，节点按区间下界排序，各节点额外记录子树内最大上界，支持高效区间搜索。每个区间节点挂着命中它的查询 ID 集合。
3. **精确且完备的表示**：Firestore 的 filter（字段与常量的等值或 `in` 比较 + 至多一个关系运算符 + 合取/析取）在区间上精确可表示，不产生假阳性。
4. **pre/post-image 差集**：无需为删除类变更单独设计路径，直接对两幅镜像各跑一次查询，差集即可分别得到「新命中」与「不再命中」的查询。
5. **能力设计协同**：「至多一个关系运算符」这一 filter 限制不是妥协，而是让区间表示成立的前提；能力增强会使 $O(\log \text{queries})$ 退化。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[QueryMatcher]]
- [[Interval Tree]]
- [[Trie]]
- [[Realtime Query]]
