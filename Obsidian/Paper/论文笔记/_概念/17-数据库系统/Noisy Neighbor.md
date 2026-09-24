---
type: concept
aliases: [Noisy Neighbor, Noisy Neighbor Problem, 吵闹邻居, 嘈杂邻居问题]
---

# Noisy Neighbor

## 定义

多租户系统中一个或多个高负载租户消耗过多共享资源（CPU、内存、带宽、I/O），导致同一物理或逻辑节点上的其他租户性能下降（延迟升高、错误率上升）的现象。

## 核心要点

1. **多租户的代价**：多租户提升资源利用率并实现无服务器弹性，但天然引入 noisy neighbor 风险——每个共享任务处理来自多个客户数据库的流量。
2. **Firestore 的防御机制**：每库施加宽松资源约束保证公平；预置额外容量给自动扩缩留时间；符合 [[500/50/5 规则]] 的合规流量约束；[[Isolation Pool|隔离池]]兜底。
3. **资源估算失准的风险**：当资源消耗估算模型（如以文档数估算写代价）与实际不匹配时（如超大文档消耗更多解析资源），即使合规流量也可能成为 noisy neighbor，需要重新标定阈值。
4. **大文档实时查询的特殊放大效应**：Firestore 传播完整 before/after 文档来评估谓词，大文档上的实时查询带来显著带宽与内存开销，官方建议作为反模式避免。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)

## 相关概念

- [[Performance Isolation]]
- [[Isolation Pool]]
- [[500/50/5 规则]]
