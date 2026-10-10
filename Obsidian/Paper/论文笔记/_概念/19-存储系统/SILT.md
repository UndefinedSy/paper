---
type: concept
aliases: [SILT, Small Index Large Table]
---

# SILT

## 定义

CMU 提出的内存高效、高性能单机 KV 存储引擎，综合三种数据结构（LogStore、HashStore、SortedStore）分层组织数据，在极低的内存索引开销（每 key 仅约 0.7 字节）下实现高吞吐随机读写。

## 核心要点

1. 分层设计：新写入先进入 LogStore（append-only），积累到一定规模后转为只读的 HashStore 或经排序合并进入最紧凑的 SortedStore。
2. 核心贡献是用极紧凑的内存索引结构（如 partial-key cuckoo hashing）大幅降低索引内存开销，使单机可承载更大数据集。
3. 与 [[Bitcask]] 类似，依赖后台合并/转换过程来维持数据的紧凑性和查询效率。
4. [[SOS]] 的全内存哈希索引设计理念与 SILT 一致（哈希索引提供 $O(1)$ 查找），但 SOS 用原地覆写取代 SILT 式的分层合并过程。

## 代表工作

- SILT 原始论文（Lim et al., SOSP 2011）。
- [[SOS]]：相关工作中对比指出的哈希索引系统之一。

## 相关概念

- [[Hash Index]]
- [[Bitcask]]
