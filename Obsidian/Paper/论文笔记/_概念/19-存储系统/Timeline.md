---
type: concept
aliases: [时间线, Timeline ID]
---

# Timeline

## 定义

[[Neon]] / [[Lakebase]] 存储层的基本版本管理单元：一条 timeline 是一段**独立演进的数据库历史**，拥有自己的 [[WAL]] 流、[[Layer File]] 集合与索引。一个 tenant（项目）下可有多条 timeline。

## 核心要点

1. **创建语义**：项目起始有一条 root timeline；[[Database Branching|分支]] 或 [[PITR|point-in-time restore]] 会在同一 tenant 下创建**新 timeline**。
2. **共用物理存储**：同一 tenant 的 timeline 被**合置在同一批 [[Pageserver]] 上**，因此分支能高效读取其 copy-on-write 祖先的数据；但它们的 WAL 可以复制到不同的 [[Safekeeper]]。
3. **读的递归解析**：子 timeline 只保存"父 timeline ID + branch LSN"的引用；读取时若本 timeline 层文件无法满足，递归查父 timeline 的层（元数据在其索引文件中）。**深分支层级因此损害读性能——一次读要跨更多 timeline**。
4. **操作基本独立**：压实与垃圾回收等 timeline 级操作在父子 timeline 上大体独立进行。
5. **detach**：把子分支从父分支摘离是瞬时的元数据操作；PS 异步把祖先数据拷贝到被摘离的 timeline。摘离后可重新挂到某个祖先下；也可被提升为新的 root。**Lakebase 当前不支持合并两条分支**（分歧的物理数据如何调和没有良定义）。

## 代表工作

- [[Neon]]: tenant / timeline 抽象来源。
- [[Lakebase]]: 分支、PITR、static replica 都以 timeline 为承载；正文指出在 agent 定稿后应 detach 并提升为 root 以规避深分支的读放大。

## 相关概念

- [[Database Branching]]
- [[Pageserver]]
- [[PITR]]
- [[LSN]]
- [[Layer File]]
