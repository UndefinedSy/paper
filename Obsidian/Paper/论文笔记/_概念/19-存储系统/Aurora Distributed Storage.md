---
type: concept
aliases: [Aurora 分布式存储, Aurora Storage Layer]
---
# Aurora Distributed Storage
## 定义
[[Amazon Aurora]] 的存储层：把数据库卷切成 segment，每个 segment 的每次写都复制到跨 3 个可用区的 6 个存储节点上。
## 核心要点
1. 复制布局为 6 副本（2 per AZ），据此在 quorum 层面同时容忍 AZ 级与节点级故障。
2. 可容忍「整 AZ 失效 + 另一 AZ 一个副本失效」而数据不丢；整 AZ 失效下仍保持可写。
3. 自动检测 segment 故障并用健康副本修复；容量按需弹性伸缩。
4. **单写者语义**：给定卷同一时刻只接受一个 Aurora PostgreSQL 实例的写入——[[Aurora Limitless Database]] 用它规避 [[Split-Brain]]，让 zombie 实例在写失败后自行退出。
5. 使 Limitless 得以管理跨多 shard 的 PB 级数据，且**无需额外共识协议**。
6. 备份集成：写入存储的每条日志记录都带上事务提交协议赋予的时间戳，从而支持「精确到某时间点」的全局一致备份。
## 代表工作
- [[Amazon Aurora]]：本存储层的原始设计。
- [[AuroraLimitless]]：router 与 shard 各自的卷都托管于此。
## 相关概念
- [[Amazon Aurora]]
- [[Copy-on-Write Cloning]]
- [[Split-Brain]]
- [[Lease]]
- [[Log as Data]]
- [[WAL]]
