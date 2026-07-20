---
type: concept
aliases: [LavaStore, Lava Store, ByteDance存储引擎]
---

# LavaStore

## 定义

字节跳动自研的高性能、低成本本地存储引擎，专为云服务设计，作为 ABase 等系统的底层 DataNode 存储层。

## 核心要点

1. 针对 ByteDance 超大规模云服务场景优化，强调性能和成本效益
2. 支持 compaction 和 garbage collection 操作（类 LSM-tree 架构）
3. ABase 中通过 RU 写上限控制 LavaStore compaction 过程中的延迟抖动
4. VLDB 2024 发表

## 代表工作

- Wang et al., VLDB 2024: "LavaStore: ByteDance's Purpose-Built, High-Performance, Cost-Effective Local Storage Engine for Cloud Services"
- [[ABase]]: DataNode 底层存储引擎

## 相关概念

- [[LSM-tree]]
- [[SSTable]]
- [[WAL]]
