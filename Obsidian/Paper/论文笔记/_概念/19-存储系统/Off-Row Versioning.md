---
type: concept
aliases: [off-row 版本, Off-row Versioning]
---

# Off-Row Versioning

## 定义

[[Persistent Version Store]] 的一种形态：把超出 [[In-Row Versioning|in-row]] 限制的旧版本存入一张全库共享、面向 append-only 优化的内部无索引表，通过 (Page ID, Slot ID) 物理定位查找。

## 核心要点

1. 每个数据库维护单一 off-row PVS 表，存放所有用户表产生的版本；每条版本是该表中一行，含元数据列与保存完整版本内容的通用二进制列，与源表 schema 无关。
2. 表被设计为 append-only，只分配新页存储新版本，因此清理任务可以按页粒度跟踪版本状态、整页回收，而不必逐行清理。
3. 每个 off-row PVS 页维护 `PageMaxXdesId`（利用 XdesID 单调递增只存最大值），配合 `OldestActiveXdesId`、`OldestAbortedXdesId`、[[MinUsefulXTS]] 判断整页是否可回收。
4. off-row PVS 页可通过 SQL Azure 原生支持的「按内部表定位所属页」操作被发现，恢复时也据此重建 `PVSPageTrackerList`。

## 代表工作

- [[AzureMVCCVersionCleaner]]：论文的主要讨论对象，全部清理优化均围绕 off-row PVS 页展开（同时适用于 in-row）。

## 相关概念

- [[In-Row Versioning]]
- [[Persistent Version Store]]
- [[MinUsefulXTS]]
