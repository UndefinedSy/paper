---
type: concept
aliases: [Bigtable 2006, Original Bigtable Paper, OSDI 2006 Bigtable]
---

# Original Bigtable Paper

## 定义

2006 年 OSDI 上发表的 "Bigtable: A Distributed Storage System for Structured Data" 论文，奠定了 Bigtable 的核心架构设计，获得超过 8000 次引用。

## 核心要点

1. 提出 Bigtable 的三维有序映射数据模型 `(row, column, timestamp) → value`
2. 定义了 tablet/tablet server/master 的集群架构
3. 使用 [[WAL]] + [[LSM-tree]] 存储引擎和 [[SSTable]] 文件格式
4. 设计了 minor/merge/major 三种 compaction 机制
5. 数据存储在分布式文件系统 [[Colossus]]（原 GFS）上
6. 元数据自引用存储，启动元数据在 [[Chubby]] 中
7. 获得 8000+ 引用，启发了 [[HBase]]、[[Cassandra]] 等系统

## 代表工作

- Bigtable: A Distributed Storage System for Structured Data (OSDI 2006, Chang et al.)

## 相关概念

- [[Bigtable]]
- [[LSM-tree]]
- [[SSTable]]
- [[WAL]]
- [[Chubby]]
- [[Colossus]]
