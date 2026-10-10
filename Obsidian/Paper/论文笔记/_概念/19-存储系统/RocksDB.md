---
type: concept
aliases: [RocksDB]
---

# RocksDB

## 定义

Facebook 基于 Google LevelDB 深度优化的高性能嵌入式 KV 存储引擎，采用经典 [[LSM-tree]] 架构，是业界事实上的磁盘 KV 存储标准实现，被广泛用作其他分布式数据库的底层存储引擎。

## 核心要点

1. LSM-tree 架构：写入先进内存 memtable，刷盘生成不可变 SSTable，多层 SSTable 通过 compaction 合并，典型写放大 10–30×。
2. 读路径需要在多层 SSTable 中逐层查找（配合 Bloom filter 加速过滤），冷缓存读放大可达每层一次 I/O 探测，生产中有效读放大约 2–12×。
3. 丰富的可调参数（block cache 大小、compaction 策略、压缩算法等）使其能适配多种工作负载，但调优复杂度高。
4. [[SOS]] 论文选择 RocksDB（v7.0）作为唯一的性能对比基线（排除 [[Redis]] 因线程模型不可比），在 YCSB 系列负载下系统性测量 WAF、RAF、吞吐、尾延迟差异，证明哈希索引 + 原地覆写方案相对 LSM-tree 的全面优势。

## 代表工作

- RocksDB（Facebook，基于 LevelDB，Cao et al. FAST 2020 的生产工作负载研究）。
- [[SOS]]：核心对比基线，实验中在 WAF（1.00× vs. 14–16×）、RAF（1.00× vs. ≈11.6×）、吞吐（最高 12.0×）、P99 延迟（最高 26.7× 改善）上全面被 SOS 超越。

## 相关概念

- [[LSM-tree]]
- [[Hash Index]]
- [[B+ Tree]]
