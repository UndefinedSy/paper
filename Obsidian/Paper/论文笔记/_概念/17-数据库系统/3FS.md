---
type: concept
aliases: [3FS, Fire-Flyer File System]
---

# 3FS

## 定义

3FS（Fire-Flyer File System）是 DeepSeek 开源的通用分布式文件系统，可被 SGLang 等推理框架用作 KV cache 后端，代表 storage-centric 的 KV cache 构建路线。

## 核心要点

1. 本身是通用分布式文件系统而非 KV-cache 专用系统，提供全局命名空间与持久化存储能力。
2. 作为 KV cache backend 时，须承担分布式文件系统本身的元数据管理、复制等开销，访问延迟高于内存方案。
3. 优势是每 GB 成本低，可以用大量低成本 SSD/云块存储堆出大容量，适合超大 KV footprint 的冷数据层。

## 代表工作

- [[PolarKV]]：作为 storage-centric 基线，在固定预算下与 PolarKV、Mooncake 对比；因文件系统开销和缺乏本地内存复用，TTFT 普遍高于 PolarKV 47%–58%。

## 相关概念

- [[KV Cache]]
- [[HDFS]]
- [[ESSD]]
