---
type: concept
aliases: [Side List, 侧列表, 重试列表]
---

# Side List

## 定义

版本清理器用于记录因抢锁失败被跳过、或因页分裂被移入含中止行的页的专用列表，在整库 sweep 完成后由主清理线程在限定时间内循环重试。

## 核心要点

1. 页分裂搬移中止行时，执行分裂的事务负责把目标页登记进 Side List，避免清理器遗漏迁移后的行。
2. Side List 本身设有容量上限；超限时不再追加新页，该轮 sweep 被判定失效并推迟到下一轮（论文称此情况罕见）。
3. 与 [[User-Assisted Cleanup]] 配合形成两级重试机制：先靠用户事务顺带清理，剩余仍失败的页留给 sweep 末尾的 Side List 重试。
4. 生产 10 天遥测：约 376 万次 sweep 受冲突负载影响，其中约 323 万次最终成功（98.6%），主要归功于 Side List 与用户协助清理的重试逻辑。

## 代表工作

- [[AzureMVCCVersionCleaner]]：应对热点写工作负载下锁冲突的核心机制之一。

## 相关概念

- [[User-Assisted Cleanup]]
- [[Logical Revert]]
- [[Aborted Transaction Map]]
