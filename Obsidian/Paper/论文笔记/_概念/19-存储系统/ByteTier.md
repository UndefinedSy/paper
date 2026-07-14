---
type: concept
aliases: [ByteTier, 分层存储, 两阶段写]
---

# ByteTier

## 定义

基于 [[ByteStore]] 构建的分层数据管理层，引入两阶段写机制：前台写先在性能池中聚合和缓存，后台异步将数据重组并持久化到容量池，兼顾写延迟敏感性和存储效率。

## 核心要点

1. **两阶段写机制**：
   - **前台阶段**：写入数据缓存到 [[TierTable]]（内存）并持久化到性能池的 [[WAL]] 文件，快速吸收突发写
   - **后台阶段**：TierTable 满足刷盘条件后，异步将数据 flush 到容量池的 ChunkServer；后台 apply 完成后 TierTable 数据和 WAL 文件才可释放
2. **有状态代理要求**：代理层需协调写聚合、保存前台数据、追踪数据位于性能池还是已 apply 到容量池
3. **粒度**：Proxy 侧处理单元为 **group**，代理 reallocation 需迁移 TierTable 数据并重加载 WAL 文件
4. **状态复杂性**：ByteTier 有状态语义是 Proxy 池化面临的核心挑战——无法像无状态代理那样自由重分配

## 相关概念

- [[ByteStore]]: 底层 DFS，提供性能池和容量池
- [[TierTable]]: ByteTier 的内存写缓冲结构，同时作为读加速组件
- [[WAL]]: 前台写的持久化载体
- [[CLAPS]]: 解决 ByteTier 有状态代理池化问题的系统

## 代表工作

- [[CLAPS]]: CLAPS: A Load-Aware Proxy Resource Pooling System, 2026 SIGMOD
