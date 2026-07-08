---
type: concept
aliases: [Discard Compaction, 丢弃压缩]
---

# Discard Compaction

## 定义

Bigtable 中一种高效的 replication log 垃圾回收技术，直接删除低复制水位以下的 replication log SSTable，无需读取数据或决定丢弃什么。

## 核心要点

1. 不读取数据内容，直接基于复制水位判断可丢弃的文件
2. 比传统需要读取并决策的压缩方式更高效
3. 专门用于复制日志的垃圾回收
4. 只读 tablet 在无更新后释放 commit log 引用，也是类似的资源优化思路

## 代表工作

- [[Bigtable]]: 复制日志垃圾回收

## 相关概念

- [[Bigtable]]
- [[LSM-tree]]
- [[SSTable]]
