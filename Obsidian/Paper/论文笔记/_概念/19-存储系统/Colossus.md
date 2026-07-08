---
type: concept
aliases: [Google Colossus, Google分布式文件系统]
---

# Colossus

## 定义

Google 的下一代分布式文件系统，是 GFS (Google File System) 的继任者，作为 [[Bigtable]] 等系统的底层存储。

## 核心要点

1. 继承 GFS 设计理念，在可扩展性和性能上大幅改进
2. [[Bigtable]] 的所有数据（SSTable、commit log）存储在 Colossus 上
3. Tablet 移动不需要数据迁移，因为数据存储在共享文件系统上
4. 支持离线访问：batch worker 可直接从 Colossus 读取 SSTable
5. 提供端到端校验支持，写入时接受校验数据

## 代表工作

- [[Bigtable]]: 底层分布式文件存储

## 相关概念

- [[Bigtable]]
- [[SSTable]]
- [[Chubby]]
