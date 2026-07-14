---
type: concept
aliases: [SPFA, Shortest Path Faster Algorithm, 最短路径快速算法]
---

# SPFA

## 定义

Bellman-Ford 算法的队列优化版本，用于在带权有向图中（包含负权边）求解单源最短路。通过维护一个队列，避免 Bellman-Ford 对所有边的冗余松弛，在实践中通常显著快于原始 Bellman-Ford。

## 算法流程

1. 初始化：源点距离为 0，其余为 +∞；将源点加入队列
2. 弹出队首节点 $u$，对 $u$ 的所有出边 $(u, v, w)$：若 $dist[u] + w < dist[v]$，则更新 $dist[v]$，并将 $v$ 加入队列（若不在队列中）
3. 重复直至队列为空
4. 若某节点入队次数超过 $|V|$，则存在负权环

## 时间复杂度

- 平均情况：$O(kE)$（$k$ 为常数，实践中通常 $k \approx 2$）
- 最坏情况：$O(VE)$（等同于 Bellman-Ford）

## 核心要点

1. **适用场景**：含负权边的图，或需要处理费用图中的最短增广路（[[MCMF]]）
2. **CLAPS 中的应用**：作为 MCMF 求解器中寻找最小费用增广路的核心算法，迭代增广直至无增广路
3. **负权处理**：在反向边（残差图）中天然存在负权，SPFA 比 Dijkstra 更适合

## 相关概念

- [[MCMF|最小费用最大流]]: SPFA 是其最短增广路的常用求解器

## 代表工作

- [[CLAPS]]: 用于 ProxyServer 机架分配的 MCMF 求解，2026 SIGMOD
