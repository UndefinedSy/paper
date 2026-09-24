---
type: concept
aliases: [Isolation Pool, 隔离池, 专属容量池]
---

# Isolation Pool

## 定义

Firestore 多租户架构中用于处理异常流量的运维手段：各组件的并行专属部署，带有独立的容量，SRE 可快速、临时地将反复违规或资源消耗估算失准的数据库流量从共享池移入隔离池，为工程团队争取分析时间。

## 核心要点

1. **作为自动机制的兜底**：只有在自动限流和负载削减无法有效处理异常流量时（例如资源消耗模型与实际不匹配）才动用，每年约 5 次。
2. **独立容量**：隔离池是共享池的并行部署，有专属机器容量，被移入的流量不再占用共享池资源，从而消除对其他租户的影响。
3. **一致性挑战**：把流量从一个 ChangeLog 池迁移到另一个时，所有下游消费者（QueryMatcher、Frontend）必须同步迁移，且迁移过程中不能丢失任何数据变更；为此 Firestore 在分片所有权机制上添加了专门的 pool change 协议。
4. **典型触发场景**：流量激增、超大文档的实时查询（before/after 文档传播带来极高带宽和内存开销）、资源估算模型失准导致的持续超额消耗。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[Performance Isolation]]
- [[Noisy Neighbor]]
- [[500/50/5 规则]]
- [[ChangeLog]]
