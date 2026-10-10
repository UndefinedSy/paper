---
type: concept
aliases: [Mooncake, KVCache-centric Serving]
---

# Mooncake

## 定义

Mooncake 是一种 memory-centric 的分布式 KV cache 架构，把多台主机的 DRAM 池化为共享远端内存，用"以更多存储换更少计算"的思路为 LLM chatbot 服务提供跨节点 KV cache 复用。

## 核心要点

1. 把 KV cache 完全放在 disaggregated DRAM 上，依赖高速互联（如 RDMA）跨主机访问，不涉及持久化存储层。
2. 优势在容量足够覆盖热集时效果接近理想（延迟低、吞吐高）；劣势是内存容量相对云块存储昂贵，大 KV footprint 场景下容量瓶颈明显。
3. 在 PolarKV 论文中作为 memory-centric 基线，与 storage-centric 的 [[3FS]] 和 PolarKV 自身的混合层级设计对照。

## 代表工作

- [[PolarKV]]：在固定云资源预算下与 Mooncake 对比，footprint 增大时 Mooncake 性能下降甚至不如 3FS，而 PolarKV 的混合层级持续领先。

## 相关概念

- [[KV Cache]]
- [[Disaggregated Memory Pool]]
- [[RDMA]]
