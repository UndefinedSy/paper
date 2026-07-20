---
type: concept
aliases: [SA-LRU, Size-Aware LRU, 尺寸感知LRU]
---

# SA-LRU (Size-Aware LRU)

## 定义

Size-Aware LRU 是标准 LRU 缓存的改进版本，在驱逐决策时同时考虑缓存项的大小和访问频率，优先驱逐"占用内存多但命中次数少"的大型数据，以提升总体缓存命中率。

## 核心要点

1. 标准 LRU 对所有大小的 KV 一视同仁，大 KV 会排挤大量小 KV 导致总体命中率下降
2. SA-LRU 在驱逐时加入大小维度权重：相同访问频率下，大 KV 优先被驱逐
3. 小 KV 访问代价低（不含磁盘 I/O），保留效益高；大 KV 缓存占用多，若非热键则性价比低
4. 适用于 KV 大小分布极度不均（0.1KB 到 5MB）的多租户场景

## 代表工作

- [[ABase]]: DataNode 层缓存策略，应对 ByteDance 多业务线 KV 尺寸多样性

## 相关概念

- [[AC-LRU]]
- [[LRU|Least Recently Used]]
