---
type: concept
aliases: [LSM 虚拟化]
---

# LSM Virtualization

## 定义

SalesforceDB 提出的架构层，利用租户聚簇存储和不可变 LSM，通过 [[Virtual File References|虚拟文件引用]]（Extent Reference / Extent Reference Set）将 LSM 中实际不可变的物理 extent 与逻辑可见的租户数据解耦，使租户克隆、迁移等操作转化为元数据级别的常数时间操作。

## 核心要点

1. **Extent Reference (ER)**：对物理 extent 的抽象层，包含 extent ID、可见键空间（minKey/maxKey）、时间戳范围、租户映射（源租户/新租户）等元数据，多个 ER 可指向同一个物理 extent
2. **Extent Reference Set (ERS)**：关联一组 ER 的集合；由于记录按租户聚簇，ERS 可被精确定义为匹配某租户在 LSM 中的全部数据
3. **[[Tenant Snapshot]]**：将一个租户的 ERS 保存入存储清单即构成快照，创建/导出/导入/删除均为元数据操作，无需移动或拷贝物理数据
4. **跨数据库共享的前提**：需要通过 schema fingerprint 比对确保 schema 兼容，且 extent 只有在不再被任何数据库的任何 ERS 引用时才会被删除
5. 支撑了租户迁移（Tenant-Migration）和沙箱创建（Sandbox Creation）两大应用场景；后者额外引入源→克隆租户的映射，在查询和 compaction 时做运行时租户翻译
6. 该机制只有在存储不可变的前提下才成立——虚拟文件引用之所以有意义，是因为它们指向的文件永不改变

## 代表工作

- [[SalesforceDB]]：LSM Virtualization 的唯一代表工作，是本文最核心的架构创新

## 相关概念

- [[Virtual File References]]
- [[Tenant Snapshot]]
- [[Immutability]]
- [[LSM-tree]]
- [[Tenant Encryption]]
- [[Snowflake]]
