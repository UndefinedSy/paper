---
type: concept
aliases: [Salesforce Database Store]
---

# SDBStore

## 定义

Salesforce Database Store，SalesforceDB 所依赖的高度分布式、不可变存储系统，fork 自 [[Apache BookKeeper]]，以云对象存储（如 [[AWS S3]]）为最终系统记录，叠加分布式缓存层提供低延迟访问。

## 核心要点

1. **双缓存架构**：数据缓存（write-through，基于 ephemeral NVMe，优化高吞吐顺序写与低延迟随机读）与事务日志缓存（write-back，基于 [[AWS EBS]] 类持久块存储，优化低延迟持久写与高吞吐流式读），二者共享 API 但独立实现、独立扩展
2. **存储模型**：以 Extent（类似文件，128-bit UUID 标识，单写者，创建→追加 fragment→关闭→删除的生命周期）和 Fragment（类似 block，不可变、变长、按序号编号）组织
3. 单个 SDBStore 缓存集群可容纳约 1PB 数据，可被多个 SalesforceDB 实例共享
4. 日志缓存采用 ensemble 式 quorum（$E_n=6, Q_w=6, Q_a=4, Q_r=3$，跨 3 AZ），实现低尾延迟同时保证故障转移时的强一致性
5. 文件到节点元数据由 BookKeeper 实现、存储在 [[ZooKeeper]] 中；extent 放置采用 zone-aware 加权算法

## 代表工作

- [[SalesforceDB]]：唯一代表工作，SDBStore 是其不可变分布式存储基础

## 相关概念

- [[Apache BookKeeper]]
- [[AWS S3]]
- [[AWS EBS]]
- [[Immutability]]
- [[LSM-tree]]
- [[Hedged Reads]]
