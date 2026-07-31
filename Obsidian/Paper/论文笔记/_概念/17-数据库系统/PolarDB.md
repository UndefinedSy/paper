---
type: concept
aliases: [阿里云 PolarDB]
---
# PolarDB
## 定义
阿里云的云原生关系数据库，走过从 scale-up 到 scale-out 的演进并达到 20 亿 tpmC。
## 核心要点
1. commit 时间戳有两种方案：**RDMA-based timestamp oracle** 或 [[Hybrid Logical Clock]]。
2. HLC 方案延迟更低，但以**放松一致性保证**为代价。
3. timestamp oracle 方案一致性强，但引入中心组件（靠 RDMA 压低其延迟）。
4. [[Aurora Limitless Database]] 选择第三条路：与有界误差时钟服务深度集成，既无中心 oracle 也不放松一致性。
## 代表工作
- From scale-up to scale-out: PolarDB's journey to achieving 2 billion tpmC (VLDB 2025)。
## 相关概念
- [[Hybrid Logical Clock]]
- [[Sharding]]
- [[Shared-Storage Architecture]]
