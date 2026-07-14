---
type: concept
aliases: [MCMF, Minimum-Cost Maximum-Flow, 最小费用最大流, 最小费用流]
---

# 最小费用最大流

## 定义

网络流优化问题：在有向图中找到一个最大流的同时，使总费用最小。每条边有容量上限和单位流量费用，目标是在满足流量守恒和容量约束的前提下，最大化源点到汇点的流量并最小化总费用。

## 数学形式

设有向图 $G = (V, E)$，源点 $S$，汇点 $T$，每条边 $(u,v)$ 有容量 $c(u,v)$ 和费用 $w(u,v)$：

$$
\min \sum_{(u,v) \in E} w(u,v) \cdot f(u,v)
$$

$$
\text{s.t.} \quad \sum_v f(u,v) - \sum_v f(v,u) = \begin{cases} F & u = S \\ -F & u = T \\ 0 & \text{otherwise} \end{cases}, \quad 0 \le f(u,v) \le c(u,v)
$$

## 核心要点

1. **求解方法**：通常通过反复在残差图中寻找最短（最小费用）增广路来求解
2. **常用算法**：
   - [[SPFA]] + 增广路（Bellman-Ford 变体，适合一般费用图）
   - Dijkstra + 势函数（Johnson 算法，适合无负权边）
   - Cost Scaling 算法（适合大规模实例）
3. **CLAPS 中的应用**：将 Proxy 机架分配建模为 MCMF 问题，流网络节点包含 Source、cluster-front、cluster-back、rack、free-proxies、Sink，用于同时优化分散性和亲和性两个目标

## CLAPS 中的流网络结构

- $S \to$ cluster-front($i$)：容量 MinRacks($i$)，费用 0（分散约束）
- cluster-front 节点互连：无限容量，费用 0（跨机架分配）
- cluster-front($i$) → cluster-back：容量无限，费用随 level 递增（分散惩罚项）
- cluster-back → rack($j$)：容量 1，费用 $\psi \cdot \text{hash}(i,j)$（亲和性项）
- rack($j$) → free-proxies：容量 1，费用递减（利用机架容量激励）
- free-proxies → $T$：无限容量，费用 0

## 相关概念

- [[SPFA]]: MCMF 中用于寻找最短增广路的算法
- [[Proxy 资源池化]]: CLAPS 利用 MCMF 解决 ProxyServer 机架分配问题

## 代表工作

- [[CLAPS]]: 将 Proxy 机架分配建模为 MCMF 问题，2026 SIGMOD
