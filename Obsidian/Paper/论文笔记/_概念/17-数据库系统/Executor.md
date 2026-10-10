---
type: concept
aliases: [Executor Node]
---

# Executor

## 定义

SalesforceDB 扩展架构中的对等计算节点，接受读写应用连接，tail [[Coordinator]] 的事务日志以在本地维护数据视图，并独立完成大部分查询规划与执行，仅在必要时通过 RPC 与 Coordinator 交互。

## 核心要点

1. 本地维护从持久化最大 XCN 到自身 **redoXCN**（已完全重放进本地 Memstore 的最近提交时间戳）之间的完整数据视图
2. 通过 **Memstore Bloom Filter**（从 Coordinator 获取）判断目标数据是否可能存在于 Coordinator 的未持久化 Memstore 中，避免不必要 RPC——实测消除约 99% 的 probe RPC 和 60–70% 的 scan RPC
3. 为保证未提交事务本地可读自己的写入，Executor 也将事务写应用到本地 Memstore，并在 redo 过程中与 Coordinator 对账
4. 约 99.9% 的查询数据可通过共享不可变持久化存储本地读取，无需任何跨节点协调
5. 读写请求可被路由到任意 Executor，而非按 keyspace 分区，从而支持不可分区的高倾斜工作负载

## 代表工作

- [[SalesforceDB]]：与 [[Coordinator]] 构成非对称扩展架构的另一半，是实现近线性横向扩展（16 节点 11.79× 容量增益）的关键

## 相关概念

- [[Coordinator]]
- [[Bloom Filter]]
- [[Transaction Commit Number]]
- [[Read Committed Snapshot Isolation]]
