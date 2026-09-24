---
type: concept
aliases: [ATM, 中止事务映射表]
---

# Aborted Transaction Map

## 定义

SQL Azure 中记录所有已中止事务 XdesID 的数据结构，事务中止时把自己的 XdesID 加入其中，版本清理器据此判断某行版本是否属于需要 [[Logical Revert|逻辑回滚]] 的中止事务。

## 核心要点

1. 每轮清理 sweep 开始前先复制当前 ATM 快照，确保清理器不会误移除 sweep 开始后才中止的事务（这些事务的版本本轮不保证被清理）。
2. sweep 成功完成对应中止事务的所有逻辑回滚后，才把这些事务从 ATM 中移除。
3. `OldestAbortedXdesId`（ATM 中最小的 XdesId）是 [[Persistent Version Store]] 页清理判据的三重水位之一；ATM 移除延迟会直接拖慢该水位推进。
4. [[Transaction-Level Version Tracking]] 引入后，只有页面确实混有 ATM 中中止事务的版本时才需要继续依赖该水位。

## 代表工作

- [[AzureMVCCVersionCleaner]]：以 ATM 为基础实现逻辑回滚与中止事务水位追踪。

## 相关概念

- [[Logical Revert]]
- [[MinUsefulXTS]]
- [[Persistent Version Store]]
