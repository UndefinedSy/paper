---
type: concept
aliases: [AC-LRU, Active-Update LRU, AU-LRU, 主动更新LRU]
---

# AC-LRU (Active-Update LRU)

## 定义

Active-Update LRU 是代理层缓存策略，在标准 LRU 基础上增加主动刷新机制：当缓存项临近过期（TTL 到期）前，自动从后端更新数据，防止缓存到期导致的请求尖峰。

## 核心要点

1. 标准缓存到期时，所有等待该键的请求同时穿透到后端（Cache Stampede / Thundering Herd 问题）
2. AC-LRU 在键接近过期时提前异步刷新，消除缓存到期时的流量尖峰
3. 结合 [[Limited Fan-out Hash Routing]] 使用，将代理分组，每组代理维护独立缓存，提升总体命中率
4. 代理缓存命中时直接返回，不消耗 DataNode 资源，也不计 RU 费用

## 代表工作

- [[ABase]]: Proxy 层缓存策略，专为热键场景设计

## 相关概念

- [[SA-LRU]]
- [[Limited Fan-out Hash Routing]]
