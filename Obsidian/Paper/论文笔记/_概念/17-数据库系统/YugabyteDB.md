---
type: concept
aliases: [YB]
---
# YugabyteDB
## 定义
开源的分布式 SQL 数据库，PostgreSQL 兼容层 + 分片复制存储层。
## 核心要点
1. 同时支持 serializability 与 [[Snapshot Isolation]]，隔离级别覆盖面比 [[CockroachDB]] 更广。
2. 同样使用 [[Hybrid Logical Clock]] 确定 commit 时间戳。
3. **局限：HLC 有时需要「等出集群内最大时钟偏移」**；在缺少 [[TrueTime]] 或 [[Amazon Time Sync]] 这类高精度时钟服务时，该值必须保守地设到**数百毫秒**，直接抬高事务延迟。
4. 这恰好凸显了 [[Aurora Limitless Database]] 依托 <1ms 误差界时钟服务的价值。
## 代表工作
- YugabyteDB: Fundamentals of distributed transactions（官方文档）。
## 相关概念
- [[Hybrid Logical Clock]]
- [[Snapshot Isolation]]
- [[CockroachDB]]
- [[Amazon Time Sync]]
