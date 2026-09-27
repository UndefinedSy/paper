---
type: concept
aliases: [对象存储, Cloud Object Storage, S3]
---

# Object Storage

## 定义

云端提供的**高持久、低成本、可水平扩展的键值式存储服务**（如 AWS S3）。它以"先写后读、对象不可变、按 API 调用与容量计费"为特征，是第三代数据库把持久层外置化的物质基础。

## 核心要点

1. **持久性**：AWS S3 宣称 11 个 9 的持久性，跨可用区复制，自动按访问模式分层（冷数据更便宜、热数据仍快），并且**版本化、支持恢复**——可防御软件错误。
2. **成本结构**：与计算差价悬殊。AWS EC2 R8gd.4xlarge 按需约 $1.18/hr，而 10 GB S3 Standard 约 $0.0003/hr，**相差三个数量级**；PUT/GET 请求费与 compaction 分层费在重摄入下仍相对计算很小。因此按 vCPU 吞吐可作为服务成本效率的代理。
3. **延迟约束**：对象存储单独**无法**满足 OLTP 的毫秒级响应，因此必须叠加存储层级：计算本地 SSD 缓存 → [[Pageserver]] 物化并缓存 → 冷键才回源对象存储。
4. **开放格式的价值**：存储格式开放（标准 [[PostgreSQL]] 页 + 湖格式）使外部引擎可绕过主引擎直接读取，这是"多引擎访问"与"消除结构性厂商锁定"的前提。
5. **写放大与请求成本**：小对象高频写会产生请求费并提供差吞吐，因此 [[Safekeeper]] 倾向**批量上传** WAL 段（典型 16 MB），[[Pageserver]] 则把层文件成批上传。
6. **作为恢复源**：PS 的 remote consistent LSN 与 SK 的 backup LSN 都是"已在对象存储中"的水位；本地盘丢失或首次服务时从这里恢复。

## 代表工作

- [[Lakebase]]: 全部数据（**含 WAL**）持久化在对象存储，是"第三代数据库"的核心判据。
- [[Amazon Aurora]]: 第二代做存储分离但仍用私有存储层而非通用对象存储。

## 相关概念

- [[Safekeeper]]
- [[Pageserver]]
- [[Layer File]]
- [[Disaggregated Storage-Compute Architecture]]
- [[Lakehouse]]
