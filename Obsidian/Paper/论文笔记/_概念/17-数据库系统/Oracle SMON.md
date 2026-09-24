---
type: concept
aliases: [Oracle SMON, System Monitor]
---

# Oracle SMON

## 定义

Oracle 数据库中负责清理 undo 空间的后台进程（System Monitor），在用户事务同步完成回滚后，回收不再被任何人需要的旧版本所占用的 undo 空间。

## 核心要点

1. Oracle 的回滚由用户事务自身同步完成（与 InnoDB 类似），SMON 只在此之后清理 undo 空间。
2. Oracle 采用「undo 保留策略」而非主动删除：数据库尽量为 undo 记录保留一个最小时长，超过保留期且无人需要才允许被新 undo 记录覆写。
3. 该策略的副作用是长查询可能遭遇 undo 被提前覆写（ORA-01555 "snapshot too old"），导致事务被终止；SQL Azure 不设保留期，虽然长事务会增大版本存储，但不会终止事务。
4. Oracle 版本仅维护 diff（类似 InnoDB），且只存 off-row，不像 SQL Azure 同时使用 in-row 与 off-row。

## 代表工作

- [[AzureMVCCVersionCleaner]]：Related Work 中与之对比 undo 保留策略与终止事务的取舍。

## 相关概念

- [[MVCC]]
- [[Persistent Version Store]]
- [[Logical Revert]]
