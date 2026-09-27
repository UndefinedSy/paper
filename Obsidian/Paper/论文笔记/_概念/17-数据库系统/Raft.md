---
type: concept
aliases: [Raft 协议]
---
# Raft

## 定义
Raft 是一种用于管理**复制日志**的共识算法，通过 leader 选举、日志复制与安全性约束三部分，使一组副本对日志顺序达成一致。

## 核心要点
1. **强 leader 设计**：所有客户端请求经 leader 处理，leader 把日志条目复制到多数派（quorum）后即可提交。相比 Paxos 更易理解与实现，因此被大量工业系统采用。
2. **成员多数派是安全前提**：任一时刻至多一个 leader 能获得多数派支持，从而避免 [[Split-Brain]]。但**只有存储层配合 fencing 才能防止旧 leader 继续写**。
3. **[[K2Plus]] 的批评点**：[[CockroachDB]] 用 **per-range Raft group** 组织数据，副本跨区域时，分区操作（split/merge/migration）可能涉及全局协调或全局元数据的同步更新，**协调开销随事务涉及的 range 数增长**，限制区域自治与倾斜负载下的响应能力。
4. **K2+ 的替代路线**：用 [[TrueTime]] 提供全局序，把「复制」与「range leadership」分离，复制中间件可在单个 replication view 内复制来自不同 range 的数据，事务协调者跟踪复制进度的开销更低。论文也提到 [[TiDB]] 与 CockroachDB 在同步部署下都实现 Raft。

## 代表工作
- [[CockroachDB]]：per-range Raft 的工业代表。
- [[TiDB]]：以 Raft 复制 TiKV region 的 HTAP 数据库。
- [[K2Plus]]：明确选择不用 per-range 共识，改以中间件复制 + TrueTime 排序。

## 相关概念
- [[CockroachDB]]
- [[TiDB]]
- [[Split-Brain]]
- [[Leader Election]]
- [[Seal]]
