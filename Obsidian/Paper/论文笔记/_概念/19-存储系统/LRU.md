---
type: concept
aliases: [Least Recently Used, 最近最少使用]
---
# LRU
## 定义
淘汰最长时间未被访问对象的缓存替换策略，以近期访问作为未来复用概率的近似。
## 核心要点
1. 实现简单，适合具有时间局部性的工作负载。
2. 大范围顺序扫描可能污染缓存。
3. 多优先级系统通常需要分池或加权版本保护关键数据。
## 代表工作
- [[TokaDB]]：Serving Data Pool 使用 LRU，training 数据被隔离到 FIFO Staging Pool。
- [[ABase]]：使用 AC-LRU 与 SA-LRU 适配主动更新和多尺寸 KV。
## 相关概念
- [[Hierarchical Cache Allocator]]
- [[AC-LRU]]
- [[SA-LRU]]
