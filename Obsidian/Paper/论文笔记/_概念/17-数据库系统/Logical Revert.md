---
type: concept
aliases: [逻辑回滚, Undo]
---

# Logical Revert

## 定义

将某行被中止事务修改后留下的旧版本重新恢复到数据页主位置的过程，是版本清理器执行 undo 的具体手段，与 [[Constant Time Recovery]] 的异步 undo 模型配套。

## 核心要点

1. 清理器对页/表加意向排他锁，用行上 XdesID 与 [[Aborted Transaction Map|ATM]] 比对判断该行是否被中止事务修改。
2. 若匹配则执行逻辑回滚，把最近一个已提交版本恢复到主位置；同时顺带清理不再需要的 in-row 版本。
3. 依赖版本链短路指针（见 [[AzureMVCCVersionCleaner]] Figure 2）可将遍历深度从链长压缩到常数（≤3）。
4. 必须在单轮 sweep 内完成某中止事务关联的所有回滚，否则该事务无法从 ATM 中移除，进而拖慢 OldestAbortedXdesId 推进。

## 代表工作

- [[AzureMVCCVersionCleaner]]：以短路指针、Side List、User-Assisted Cleanup 加速和保障逻辑回滚的完整性。

## 相关概念

- [[Persistent Version Store]]
- [[Constant Time Recovery]]
- [[Side List]]
