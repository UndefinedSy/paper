---
type: concept
aliases: [delta-base architecture, delta-base 双层架构]
---

# Delta-Base Architecture

## 定义

把实时写入路径（delta 层）与主索引结构（base 层）解耦的双层存储架构：写入先落到轻量的 delta 结构，再由后台合并进基索引。

## 核心要点

1. **三大组件**：[[Delta File|delta 文件]]（承接实时写）、base index（主索引结构，形态随索引类型变化）、[[WAL]]（故障恢复）。
2. **触发时机**：后台进程监控 delta 文件，当累积向量数超过阈值**或**经过特定时间间隔时触发 `flush`。
3. **三重收益**：低延迟写入（环形队列 + 轻量追加）、数据写后立即可见（双路径召回）、基索引无锁查询（写写冲突被限制在 delta 内）。
4. **持久化配合**：需要两级 [[WAL]]（meta 与 data 分离刷盘顺序）来在 `mmap` 语义下保证一致性——总是先刷 data WAL 再刷 meta WAL。
5. **可类比的设计**：与 LSM-tree 的 memtable/sstable 分层、以及 FreshDiskANN 的内存 delta 图 + SSD 基图有相似动机，但 Nova 的目标是系统级（mmap 文件管理、并发建索引、无锁查询扫描）而非纯算法优化。

## 代表工作

- [[Nova]]: 以该架构同时支撑 [[Nova Memory]] 与 [[Nova Disk]]，并把 `CREATE INDEX CONCURRENTLY` 的写入卸载到 delta 文件以压缩 DDL 阶段。

## 相关概念

- [[Delta File]]
- [[LSM-tree]]
- [[DiskANN]]
- [[WAL]]
