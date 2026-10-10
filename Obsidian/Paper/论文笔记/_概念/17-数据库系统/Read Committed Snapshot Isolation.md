---
type: concept
aliases: [RCSI]
---

# Read Committed Snapshot Isolation

## 定义

一种隔离级别（缩写 RCSI），要求每条语句看到一个在语句开始时刻确定的、仅包含已提交数据的一致性快照，而非整个事务固定单一快照（后者为经典 Snapshot Isolation）。

## 核心要点

1. 与标准 [[Snapshot Isolation]] 的区别在于快照粒度：RCSI 按语句取快照，事务内不同语句可以看到不同（更新的）快照
2. 基于 [[MVCC]] 实现，读操作通过比较记录的 commitXCN 与语句的 snapshotXCN 判断可见性
3. 在 SalesforceDB 中，除 SELECT FOR UPDATE 外的查询几乎不需要 RPC 到 [[Coordinator]]，除非需要访问超出本地 redoXCN 的极新近提交数据

## 代表工作

- [[SalesforceDB]]：所有 Salesforce 查询默认以 RCSI 运行，是其读路径免协调设计（约 99.9% 数据本地读取）的隔离级别基础

## 相关概念

- [[Snapshot Isolation]]
- [[MVCC]]
- [[Executor]]
- [[Transaction Commit Number]]
