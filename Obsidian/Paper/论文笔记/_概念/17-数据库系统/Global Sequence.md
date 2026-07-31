---
type: concept
aliases: [全局序列, 分布式序列]
---
# Global Sequence
## 定义
在分布式数据库中生成全局唯一（通常还要求单调）序列号的机制，典型用于自增主键。
## 核心要点
1. 朴素实现是每次取号都访问一个中心节点，会立刻成为吞吐瓶颈。
2. 主流优化是**号段预分配（range/block allocation）**：中心节点一次分配一大段号码给各前端节点，前端本地发号。
3. [[Aurora Limitless Database]] 的做法：CREATE SEQUENCE 时选一个 shard 持久化序列对象，然后向所有 router 广播初始化命令并分配**互不相交的号段**；router 本地发号，号段用尽才回访那个 shard。
4. 代价：号码在全局上不再严格递增（不同 router 的号段交错），只保证唯一性。
## 代表工作
- [[AuroraLimitless]]：号段预分配式全局序列。
## 相关概念
- [[Sharding]]
- [[AuroraLimitless]]
