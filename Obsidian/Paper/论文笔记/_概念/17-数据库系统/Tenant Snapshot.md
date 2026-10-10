---
type: concept
aliases: [租户快照]
---

# Tenant Snapshot

## 定义

将一个租户的 Extent Reference Set（ERS）保存进数据库实例的存储清单（storage manifest），即构成该租户在某一时间点的稳定、不可变数据表示；除 ERS 外还包含数据库状态元数据（如 schema fingerprint）。

## 核心要点

1. 因为底层 extent 本身不可变，租户快照自然保证了"某时刻租户数据的稳定快照"这一语义，无需额外的 MVCC 版本管理
2. 创建、导出、导入、删除租户快照均为元数据级操作，底层数据 extent 不需要移动或拷贝，由于元数据规模比数据小几个数量级，这些操作实质上是常数时间
3. 是 [[LSM Virtualization]] 支撑租户迁移（Tenant-Migration）和沙箱创建（Sandbox Creation）两大应用的核心数据结构
4. 快照中携带的 schema fingerprint 用于跨数据库实例导入时的 schema 兼容性校验

## 代表工作

- [[SalesforceDB]]：生产环境中租户快照机制支撑了 2026 年 4–6 月 89,163 次沙箱创建和 6,596 次零拷贝租户迁移

## 相关概念

- [[LSM Virtualization]]
- [[Virtual File References]]
- [[Snowflake]]
