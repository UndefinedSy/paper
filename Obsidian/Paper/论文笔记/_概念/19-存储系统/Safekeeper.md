---
type: concept
aliases: [SK, Safe Keeper, 安全守护者]
---

# Safekeeper

## 定义

[[Neon]] / [[Lakebase]] 存储层中负责 **[[WAL]] 共识复制与持久化**的组件：接收 PostgreSQL primary 广播的 WAL 记录，在本地 segment 文件刷盘，并把 quorum 已提交的段主动上传到[[Object Storage|对象存储]]。它是写路径上唯一的持久性裁决者。

## 核心要点

1. **类 [[Raft]] 复制**：PostgreSQL primary 充当 WAL proposer（leader），SK 是 acceptor。一条 WAL 记录在 quorum 写盘后才算持久，primary 的 commit LSN 取 quorum flush LSN。
2. **term 防脑裂**：SK 维护单调递增的 term，只接受见过的最高 term。新 primary 必须递增 term 并向 quorum 自我提名才能写，检测到 term 变化即须重新竞选。参见 [[Split-Brain]]。
3. **维护三条 [[LSN]]**：flush LSN（刷盘点）、commit LSN（由 primary 通知，SK 只向其下的数据流给 [[Pageserver]]）、backup LSN（已上传对象存储的水位，其下的本地段可删）。
4. **上传选举**：多个 SK gossip 各自的 commit LSN，确定性地选出一个 upload leader，并跨 timeline 做负载均衡，避免重复上传。
5. **元数据只走 Shard-0**：SK 解码每条 WAL 记录影响的 page id，只路由到相关 Pageserver；relation size 等元数据由特殊分片 Shard-0 独占跟踪。
6. **背压**：当写入快于 PS 摄入时，PS 通过 SK 上报三个 LSN（已摄入 / 已落本地盘 / 已上传对象存储），proposer 在任一水位落后 commit LSN 超阈值时限流——这个阈值直接权衡写吞吐、读延迟与恢复时间。
7. **恢复成本低**：AZ+1 故障下，恢复一个失效 SK 通常只需复制少数几个尚未备份的 16 MB WAL 段。

## 代表工作

- [[Neon]]: SK/PS 拆分架构的源头。
- [[Lakebase]]: 每实例 3 个 SK 分布在 3 个 AZ，容忍单节点或单 AZ 故障；计算启动时"同步 SK"这一步仅约 5 ms，同时用于声明唯一 primary。

## 相关概念

- [[Pageserver]]
- [[WAL]]
- [[Raft]]
- [[Split-Brain]]
- [[LSN]]
- [[Object Storage]]
