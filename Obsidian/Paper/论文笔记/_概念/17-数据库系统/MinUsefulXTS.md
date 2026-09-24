---
type: concept
aliases: [MinUsefulXTS, 全局最小可用时间戳, MinUsefulStatementXTS]
---

# MinUsefulXTS

## 定义

SQL Azure 中标记「系统内所有快照要求保留版本」的最小时间戳水位：`MinUsefulXTS` 是所有活跃快照 MinXTS 的最小值（事务级粒度），`MinUsefulStatementXTS` 是所有 RCSI/SI 活跃语句 read XTS 的最小值（语句级粒度，更激进）。

## 核心要点

1. 任何版本，其生成事务的 CSN 若低于 `MinUsefulXTS`，就不再被任何活跃快照的 MVCC 可见性需要，可视为清理候选之一。
2. 二者都是服务器级值，随快照增删按需更新，但不超过某阈值间隔更新一次，以控制维护开销。
3. RCSI（SQL Azure 默认隔离级别）在语句级建立快照，因此 `MinUsefulStatementXTS` 通常比 `MinUsefulXTS` 更新（更靠近当前时刻），能更早判定版本可清理，是 [[Statement-Level Cleanup]] 的核心依据。
4. 与 `OldestActiveXdesId`、`OldestAbortedXdesId` 共同构成 off-row PVS 页可回收的三重水位条件。

## 代表工作

- [[AzureMVCCVersionCleaner]]：以 `MinUsefulStatementXTS` 替代 `MinUsefulXTS` 实现语句级清理，回收约 27% 额外 PVS。

## 相关概念

- [[Persistent Version Store]]
- [[Statement-Level Cleanup]]
- [[Snapshot Isolation]]
