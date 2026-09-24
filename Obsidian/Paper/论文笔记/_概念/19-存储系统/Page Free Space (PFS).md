---
type: concept
aliases: [PFS, Page Free Space]
---

# Page Free Space (PFS)

## 定义

SQL Server / SQL Azure 中已有的系统页类型，原用于跟踪每个数据页的可用空间；版本清理为其扩展了一个版本标记 bit，指示对应数据页是否含有需要逻辑回滚或清理的行版本。

## 核心要点

1. 每个 PFS 页跟踪 8088 个数据页，用 1 字节记录每页的元数据（含版本标记 bit）。
2. 更新数据页前先设置其在 PFS 中的版本位；数据页本地也缓存同一状态，避免每次都访问 PFS 页，只在状态变化时才访问。
3. 清理器顺序扫描 PFS 页找到所有标记需要清理的数据页，是避免全库扫描的关键索引结构。
4. 结合 [[Read-Ahead]]，清理器先收集所有标记页再批量预取，减少逐页 I/O 等待。

## 代表工作

- [[AzureMVCCVersionCleaner]]：复用已有 PFS 结构定位待清理页，避免额外元数据开销。

## 相关概念

- [[Read-Ahead]]
- [[Logical Revert]]
- [[Persistent Version Store]]
