---
type: concept
aliases: [语句级清理]
---

# Statement-Level Cleanup

## 定义

利用 RCSI 逐语句建立快照的特性，把 [[Persistent Version Store|PVS]] 页可清理判据从事务级的 [[MinUsefulXTS]] 收紧为语句级的 `MinUsefulStatementXTS`，使已提交版本更早失去可见性需求而被回收。

## 核心要点

1. 仅适用于「页上所有版本来自同一事务」的场景，因为只跟踪最高 XdesId，无法确定多事务混合页的提交顺序。
2. 依赖 [[Transaction-Level Version Tracking]] 使单事务页更常见：`PVSPageTrackerList` 用一个 bit 标记该属性。
3. 清理先用页最高事务号的 CSN 与 [[MinUsefulXTS]] 比较，失败则用更激进的 `MinUsefulStatementXTS` 重试。
4. 30 天生产遥测显示在 `MinUsefulXTS` 造成 ≥50GB 持有量的场景下，该机制贡献约 27% 的 PVS 回收量。
5. 未来工作：通过额外维护 LatestXdesId 扩展到多事务混合页。

## 代表工作

- [[AzureMVCCVersionCleaner]]：提出者，与事务级 PVS 追踪协同工作。

## 相关概念

- [[MinUsefulXTS]]
- [[Transaction-Level Version Tracking]]
- [[Snapshot Isolation]]
