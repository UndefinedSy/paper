---
type: concept
aliases: [Bigtable Tablet Server, TServer]
---

# Tablet Server

## 定义

Bigtable 集群中托管 tablet 并处理数据读写请求的服务进程，按 partition 和 subpartition 分组，数量可随集群规模扩展。

## 核心要点

1. 托管 tablet 并处理用户数据读写请求
2. 分组为 partition，同 partition 再分为 subpartition
3. Client 直接从 tablet server 读取数据（不经 master）
4. 资源包括内存（memtable、block/row cache、Bloom filter）和 CPU
5. 执行 minor compaction（内存→SSTable）
6. 执行复制拉取、CDC 流式推送、coprocessor 逻辑
7. 大集群中单个 server 可托管数千个 tablet
8. 分离的 file builder job 承担最耗时的文件构建工作

## 代表工作

- [[Bigtable]]: 数据服务节点

## 相关概念

- [[Bigtable]]
- [[Tablet]]
- [[LSM-tree]]
- [[Bloom Filter]]
- [[Coprocessor]]
