---
type: concept
aliases: [Anna KVS]
---

# Anna

## 定义

一种基于 lattice（格）理论的无协调（coordination-free）分布式 KV 存储，通过把状态更新建模为 lattice 上的 merge 操作实现极致的读扩展性与去中心化，面向最终一致性语义。

## 核心要点

1. 核心思想：只要状态更新可以表示为满足幂等、交换、结合律的 lattice merge 操作，节点间无需协调即可安全合并并发更新，天然适合大规模水平扩展。
2. 读扩展性极佳——因为读请求可以路由到任意持有副本的节点而无需协调，但代价是只能提供最终一致性，不支持强一致性语义。
3. 与 [[SOS]] 的定位差异：Anna 面向可容忍最终一致性的场景追求读扩展性，SOS 面向需要强一致性（线性一致）与 TTL 语义的在线服务场景，两者的一致性目标根本不同。

## 代表工作

- Anna 原始论文（Wu et al., TKDE 2019）。
- [[SOS]]：相关工作中对比指出 Anna 缺少 SOS 场景所需的强一致性与 TTL 语义。

## 相关概念

- [[Linearizability]]
- [[Chain Replication]]
