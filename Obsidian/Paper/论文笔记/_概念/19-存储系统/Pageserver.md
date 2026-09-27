---
type: concept
aliases: [PS, Page Server, 页服务器]
---

# Pageserver

## 定义

[[Neon]] / [[Lakebase]] 存储层中负责把 [[WAL]] **物化成页**并按 [[LSN]] 提供页访问的组件。它从 [[Safekeeper]] 拉取 WAL，组织成 [[Layer File|层文件]]，只向计算暴露两个 API：`GetPage@LSN` 与 `GetBaseBackup`。

## 核心要点

1. **只有点查 API**：没有 range-scan 原语。`GetPage@LSN` 时找到更早 LSN 的最后一个整页镜像，把 delta WAL 记录重放到目标 LSN 返回；**请求会阻塞直到 PS 摄入到该 LSN**。
2. **二维存储格式**：按 page key × LSN 二维组织 [[Layer File]]——delta layer（一段 key 范围 × 一段 LSN 范围的原始 WAL 记录）与 image layer（某 LSN 上一段 key 范围的整页快照），后台做 LSM 式压实。对比 [[LSM-tree]] 的单维（key）组织。
3. **放大量级（Lakebase）**：写放大 $O(1)$，读代价 $O(I)$（$I$ 为 image threshold，限定 delta 链长度），空间放大 $O(1 + R + wP/N)$，生产实测约 7×。参见 [[Write Amplification]]。
4. **对象存储是真相源**：层文件持久化在[[Object Storage|对象存储]]，本地 SSD 只作缓存，本地盘紧张时按 [[LRU]] 淘汰，未来读再回源下载。timeline 的索引文件也放远端。
5. **维护四条 [[LSN]]**：last record LSN（内存摄入点）、flush LSN（批量落盘点，崩溃从此再水化）、remote consistent LSN（已上传远端，本地无有效 flush LSN 时的恢复来源）、以及供分支使用的 branch LSN。
6. **hot standby**：每个 timeline 保有跨 AZ 的热备，预先下载快照与热层文件，primary 故障时立即接管并从已缓存层服务 GetPage。
7. **sharding**：库变大时由 [[Storage Controller]] 把 timeline 透明且零停机地分片到多个 PS（Shard-0 承载元数据），提升 GetPage 吞吐。

## 代表工作

- [[Neon]]: PS 设计源头。
- [[Lakebase]]: 每个 timeline 一个 PS 分片 + 跨 AZ hot standby；计算 side 本地文件缓存（LFC）命中率通常 > 98.5%，掩盖了大部分 GetPage 延迟。

## 相关概念

- [[Safekeeper]]
- [[GetPage@LSN]]
- [[Layer File]]
- [[LSN]]
- [[PITR]]
- [[Write Amplification]]
- [[Object Storage]]
