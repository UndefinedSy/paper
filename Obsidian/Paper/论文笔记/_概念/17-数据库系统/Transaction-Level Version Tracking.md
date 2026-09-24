---
type: concept
aliases: [事务级 PVS 追踪, Transaction-Level PVS]
---

# Transaction-Level Version Tracking

## 定义

让每个事务自己维护其插入版本所使用的 [[Persistent Version Store|PVS]] 页列表，提交/中止后再分别转入全局「已提交」「已中止」两个 PVSPageTrackerList，从而把清理判据从「必须同时满足所有水位」下放到按页类别区分的机制。

## 核心要点

1. 事务生成新版本时优先复用自己列表中未写满的页，减轻全局页分配压力。
2. 页写满转入对应全局列表；中止事务的页若混有中止版本会置位 `HasAbortedTranVersions`。
3. 已提交列表中未置位该 bit 的页，清理判据可去掉 `OldestAbortedXdesId` 约束，不再被慢速中止事务水位拖慢；置位则仍需两个水位。
4. 崩溃恢复时通过扫描 PVS 表重建全部页集合，保守地全部放入中止列表，随后正常恢复两水位清理。
5. 生产 30 天遥测显示该机制在慢 sweep（≥4h）场景下贡献约 60% 的 PVS 回收量。

## 代表工作

- [[AzureMVCCVersionCleaner]]：核心提出者，解决"已提交版本被中止事务水位拖慢"的问题。

## 相关概念

- [[MinUsefulXTS]]
- [[Statement-Level Cleanup]]
- [[Persistent Version Store]]
