---
type: concept
aliases: []
---

# Bitcask

## 定义

Riak/Basho 提出的日志结构哈希表存储引擎：所有写入以 append-only 方式追加到日志文件，内存中维护哈希索引（key → 日志文件 offset），读操作通过索引直接定位磁盘偏移完成单次 I/O 查找。

## 核心要点

1. 写路径极简：顺序追加写，天然对 SSD/HDD 友好，写吞吐高。
2. 需要周期性 merge（compaction）清理被覆盖/删除的旧版本数据，回收 append-only log 占用的空间——这是其主要开销来源。
3. 索引全内存驻留，提供 $O(1)$ 点查，但受限于内存容量，总 key 数量受索引内存开销约束。
4. [[SOS]] 与 Bitcask 的核心差异在于空间回收策略：Bitcask 依赖周期性 merge 清理 append-only log；SOS 用 in-place overwrite（原地覆写）配合 block/chunk 布局，在写时同步回收旧 block，无需额外 merge 过程。

## 代表工作

- Bitcask 本身（Riak/Basho 技术报告，2010）。
- [[SOS]]：相关工作中对比指出其哈希索引思路与 Bitcask/SILT 一致，但用原地覆写取代周期性 merge。

## 相关概念

- [[Hash Index]]
- [[SILT]]
- [[LSM-tree]]
