---
type: concept
aliases: [PVS, 持久版本存储]
---

# Persistent Version Store

## 定义

SQL Azure 中随 [[Constant Time Recovery]] 引入的、位于用户数据库内部的旧行版本存储区，取代早期把版本存放在 TempDB 的做法，分为 [[In-Row Versioning|in-row]] 与 [[Off-Row Versioning|off-row]] 两种形态。

## 核心要点

1. in-row 版本作为 diff 嵌入当前行所在页，省去额外页读但受 diff/行大小上限约束（避免 B-Tree 页分裂）。
2. off-row 版本存入一张全库共享、append-only 优化的内部无索引表，按 (Page ID, Slot ID) 物理定位。
3. 每个 off-row PVS 页维护 `PageMaxXdesId`（该页最高事务号），配合 [[MinUsefulXTS]]、OldestActiveXdesId、OldestAbortedXdesId 判断整页是否可回收。
4. PVS 体量是 [[AzureMVCCVersionCleaner]] 全部优化围绕的核心对象。

## 代表工作

- [[AzureMVCCVersionCleaner]]：围绕 PVS 的可扩展、抗冲突、细粒度水位清理。

## 相关概念

- [[MVCC]]
- [[Constant Time Recovery]]
- [[MinUsefulXTS]]
