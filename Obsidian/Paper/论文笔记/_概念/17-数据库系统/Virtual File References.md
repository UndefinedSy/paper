---
type: concept
aliases: [虚拟文件引用, Extent Reference, ER, ERS]
---

# Virtual File References

## 定义

SalesforceDB [[LSM Virtualization]] 机制中的核心抽象，即 Extent Reference (ER)——对不可变物理 extent 的一层间接引用，包含 extent ID 及定义可见范围的元数据（keyspace、tenant、timestamp），使逻辑数据管理与物理持久化解耦。

## 核心要点

1. 每个 ER 包含：extent 的 ID；可见键空间 minKey/maxKey；时间戳范围；租户映射（old tenant → new tenant）
2. 多个 ER 可指向同一个物理 extent，通过元数据限制可见性到特定键范围、时间窗口或租户
3. 租户映射信息被嵌入数据库访问层，在查询处理和 LSM compaction 时透明执行租户 ID 翻译
4. 一组相关的 ER 构成 Extent Reference Set（ERS），可精确对应某个租户在 LSM 中的全部数据
5. 由于只是元数据而非物理数据的拷贝，对 ER/ERS 的操作（创建、裁剪、组合）都是常数时间级别，使其成为租户快照、迁移、克隆的构建基础

## 代表工作

- [[SalesforceDB]]：Virtual File References 的提出者和唯一代表工作，Figure 4 直观展示了跨 LSM 多层组合 ER 构造租户快照的过程

## 相关概念

- [[LSM Virtualization]]
- [[Tenant Snapshot]]
- [[LSM-tree]]
- [[Immutability]]
