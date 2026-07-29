---
type: concept
aliases: [分层缓存分配器, 优先级缓存分配]
---
# Hierarchical Cache Allocator
## 定义
按数据功能和请求优先级把共享缓存划分为多个隔离池，并为各池采用不同驻留和淘汰策略的缓存管理方法。
## 核心要点
1. metadata/index 可预热并 pin，保证稳定查询路径。
2. serving data 获得主要预算，避免被训练扫描污染。
3. training/background 使用独立 staging pool 利用短期局部性。
## 代表工作
- [[TokaDB]]：Metadata Pool + Serving Data Pool + Staging Pool，使吞吐提高约 63%、P99 降低约 13%。
## 相关概念
- [[LRU]]
- [[Bloom Filter]]
- [[SSTable]]
