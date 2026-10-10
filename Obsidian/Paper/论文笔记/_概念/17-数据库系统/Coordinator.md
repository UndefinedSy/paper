---
type: concept
aliases: [Coordinator Node]
---

# Coordinator

## 定义

SalesforceDB 扩展架构中的单一专职节点，负责处理所有底层数据库记录写入、维护集群唯一活跃的 Memstore（含行锁与最近提交数据），并托管充当时间戳 oracle 的 Epoch Coordinator。

## 核心要点

1. 每条未提交记录相当于一把隐式行锁，Coordinator 的 Memstore 负责裁决锁冲突
2. 写入经由 Coordinator 本地事务日志持久化到 [[SDBStore]]
3. Epoch Coordinator 维护两个单调递增的逻辑时钟：commit epoch（分配提交时间戳）与 visible epoch（分配读快照时间戳），大约每 1ms 推进一次并广播给所有节点
4. 是集群唯一的写协调单点，故障时由 [[ZooKeeper]] 从剩余 [[Executor]] 中选举新 Coordinator，典型恢复时间 < 5 秒
5. 在高写入倾斜（同一键大量并发写）场景下，Coordinator 的 CPU 利用率会成为扩展瓶颈（实测 16 节点下可达 92%+）

## 代表工作

- [[SalesforceDB]]：单 Coordinator + 多 [[Executor]] 的非对称扩展架构核心，是论文"relational execution has substantial scalable compute"经验教训的主体

## 相关概念

- [[Executor]]
- [[Transaction Commit Number]]
- [[ZooKeeper]]
- [[Snapshot Isolation]]
