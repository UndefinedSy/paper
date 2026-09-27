---
type: concept
aliases: [LFC, 本地文件缓存, 计算侧缓存]
---

# Local File Cache (LFC)

## 定义

[[Lakebase]] 计算节点上的**可伸缩自定义缓存**，用于缓存从 [[Pageserver]] 取回的页，使存储分离架构下的读性能接近本地盘上的原生 [[PostgreSQL]]。它补足了 PostgreSQL 静态大小的 shared buffers 的不足。

## 核心要点

1. **默认落盘**：即使缓存能完全放进内存，也**用操作系统 page cache 而非普通内存分配**来承载——因为在内存伸缩时普通分配会给内核分配器造成压力、更脆弱。盘支撑的缓存在伸缩操作中**同步**调整大小。
2. **伸缩联动**：LFC 大小随 CPU 规格一起放大。"工作集是否放得进缓存"是 OLTP 性能的决定因素，而存储计算分离让这件事更关键。
3. **工作集估计**：LFC 维护一个改造版 [[HyperLogLog]] 跟踪历史上访问过的唯一页数，供 autoscaler 判断缓存该多大。
4. **命中率**：论文实验中通常 > 98.5%，因此 GetPage@LSN 的延迟大部分被掩盖。若工作集超过 LFC，命中率会骤降（自动伸缩实验中曾跌破 30%）。
5. **伸缩单位**：以 1 GiB RAM 及其成比例的 CPU 份额为伸缩单元（scaling unit），线性且均匀。

## 代表工作

- [[Lakebase]]: LFC 是读路径三段（LFC → Pageserver → 对象存储）中最热的一段。
- [[Neon]]: 同一缓存设计的源头。

## 相关概念

- [[GetPage@LSN]]
- [[LwLSN Cache]]
- [[HyperLogLog]]
- [[Scale-to-Zero]]
- [[Pageserver]]
