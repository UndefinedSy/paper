---
type: concept
aliases: [键值分离, Key-Value Separation]
---
# KV Separation
## 定义
将 key/索引与大 value 分开存储，使 compaction 主要处理轻量索引、value 留在追加式日志中，从而降低写放大。
## 核心要点
1. 索引通常位于 LSM-tree，value 位于 ValueLog。
2. 优点是 compaction I/O 显著减少。
3. 代价是读取可能增加一次间接访问，且 ValueLog 需要 GC。
## 代表工作
- [[TokaDB]]：进一步按时间 segment 分离用户行为 value，并把 log 直接用于 KV Cache 主数据。
## 相关概念
- [[ValueLog]]
- [[LSM-tree]]
