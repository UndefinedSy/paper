---
type: concept
aliases: [Slicer, Auto-Sharding Framework, 自动分片框架]
---

# Slicer

## 定义

Google 内部的数据中心自动分片框架，负责根据实时负载动态切分和合并键空间分片，并将分片分配给任务，使服务能在无人工干预的情况下水平扩缩容。

## 核心要点

1. **负载驱动的切分/合并**：各任务周期性上报负载指标（如流量速率、载荷大小），Slicer 据此决定切分或合并操作；调整节奏设为分钟级，避免对短时突发过度反应。
2. **与 Firestore 的集成**：Firestore 的文档键空间按**写负载**通过 Slicer 自动切分（独立于 Spanner 按读写负载切分 tablet），每个 ChangeLog 任务组覆盖一段键范围；流量下降时分片合并以提升效率。
3. **500/50/5 规则兼容**：分片调整的 ~5 分钟窗口与「每 5 分钟最多增加 50% 流量」的合规增长规则对齐，在合规期间写流量被允许，超出部分被限流直到容量补齐。
4. **分片转移对实时查询的影响**：分片重新分配时，ChangeLog 通知 QueryMatcher，QueryMatcher 告知 Frontend，受影响查询临时退回 polling mode 再路由到新 QueryMatcher。

## 代表工作

- Atul Adya et al. Slicer: Auto-Sharding for Datacenter Applications. OSDI 2016.
- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[ChangeLog]]
- [[Sharding]]
- [[Performance Isolation]]
- [[500/50/5 规则]]
