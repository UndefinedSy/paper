---
type: concept
aliases: [RM, 区域管理器]
---
# Regional Manager

## 定义
Regional Manager（RM）是 [[K2Plus]] 中的区域级管理组件：把 [[Global Manager]] 分配的全局数据进一步切分为更小部分并映射到**本区域**的数据节点，同时维护区域内的数据放置与路由信息（Regional Range View）。

## 核心要点
1. **区域自治的关键**：RM 管理的分区元数据**不跨区域复制**，因此各区域的 split/merge/放置是纯本地操作，无需全局协调或跨区域通信。
2. **故障探测**：与区域内所有节点（frontend、data node、[[TrueTime|TTC Oracle]]）维持心跳连接；采用**保守的 5 s 心跳阈值**避免假阳性故障报告（实测这是节点 failover 的主要成本项）。
3. **高可用**：RM 跨 AZ 用 quorum 复制，故障时通过 view change 恢复（AZ 故障场景中 15 s 内被另一 AZ 的副本检测到并触发 leader promotion）。
4. **负载均衡**：节点过载（如 ≥90%）时选择拆分点、更新路由，并触发分区拆分与迁移；不健康节点会被隔离或摘除。
5. **路由缓存**：缓存全局 range view 并用 [[Epoch|global_epoch]] 校验；epoch 不匹配则刷新后再路由。

## 代表工作
- [[K2Plus]]：与 GM 组成两级分区管理层，实现区域本地分区自治。

## 相关概念
- [[Global Manager]]
- [[Replication Middleware]]
- [[Physical Partition]]
- [[Partition Migration]]
- [[Partition Split]]
