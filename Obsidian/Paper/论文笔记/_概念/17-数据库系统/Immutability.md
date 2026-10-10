---
type: concept
aliases: [不可变性, Immutable Storage]
---

# Immutability

## 定义

系统设计原则：数据一旦写入便不再被原地修改，任何逻辑变更都通过写入新版本实现。由 Pat Helland 在 2015 年提出其作为数据库/分布式系统设计基石的价值（"Immutability changes everything"）。

## 核心要点

1. **消除并发/损坏问题的根源**：不可变存储天然避免了可变引擎中 B-tree 分裂、页内修改引发的并发竞争与物理损坏
2. **简化缓存一致性**：缓存对象永不因写入而失效，使得分布式缓存可以极度激进地复用，无需跨节点失效协议
3. **与云对象存储天然契合**：S3 等云存储的 write-once 语义与不可变存储模型完全对齐
4. **是跨节点协调最小化的基础**：多个节点/实例可以无协调地共享不可变数据，是 scale-out 和多租户能力的共同基石
5. **派生计算可一次性完成并永久复用**：如 Bloom filter、range filter、校验和、加密等，在 flush/compaction 时异步计算一次，之后无需重算
6. 代价是读放大（read amplification）等 LSM 固有的读性能挑战，需要额外的过滤优化来缓解

## 代表工作

- [[SalesforceDB]]：将不可变性作为贯穿存储、缓存、[[LSM Virtualization]]、[[Tenant Encryption]]、协调最小化的统一设计原则，论文"经验教训"一节专门总结其跨层价值
- [[LSM-tree]]：不可变性在存储引擎层面的经典实现

## 相关概念

- [[LSM-tree]]
- [[SDBStore]]
- [[LSM Virtualization]]
- [[Bloom Filter]]
