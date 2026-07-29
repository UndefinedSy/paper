---
type: concept
aliases: [Hybrid Transactional/Analytical Processing, 混合事务分析处理]
---

# HTAP

## 定义

Hybrid Transactional/Analytical Processing，在同一数据库系统中同时支持低延迟事务操作和分析查询，通过请求优先级等机制实现工作负载隔离。

## 核心要点

1. 事务操作通常为单行读，要求低延迟
2. 分析操作为复杂多行查询，结果集大
3. [[Bigtable]] 通过请求优先级标注实现 HTAP 负载隔离
4. 优先级标注为重要流量带来更高吞吐和更低延迟
5. 复制也可用于隔离高吞吐分析流量与低延迟事务流量

## 代表工作

- [[Bigtable]]: 请求优先级支持 HTAP 工作负载
- [[ByteGraph-Dione]]: 在同一生产级图上通过轻量事务、自适应行列格式和热点副本支持动态 OLTP/OLAP 混合负载

## 相关概念

- [[Bigtable]]
- [[Eventual Consistency]]
- [[MVCC]]
- [[Snapshot Isolation]]
- [[Adaptive Dual-Format Graph Storage]]
