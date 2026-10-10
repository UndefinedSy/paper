---
type: concept
aliases: [DMP, Distributed Memory Pool, 分布式内存池]
---

# Disaggregated Memory Pool

## 定义

分布式内存池（DMP）是一种把多台 VM 的 DRAM 聚合为可通过 [[RDMA]] 远程访问的共享内存资源的架构，由 Home Node（元数据协调）与 Slab Node（物理内存承载）组成。

## 核心要点

1. **Home Node（HN）**：负责资源管理与元数据协调，不直接处理大数据流量。
2. **Slab Node（SN）**：贡献实际物理内存资源，推理引擎/客户端通过 RDMA 直连 SN 读写数据。
3. 该架构最早在云原生数据库场景（如 [[PolarDB-MP]]）中提出，用于在多主架构下共享内存，后被扩展到 LLM KV cache 场景。

## 代表工作

- [[PolarDB-MP]]：分布式内存池最初用于多主云原生数据库的共享内存设计。
- [[PolarKV]]：复用团队既有的 DMP 技术作为 KV cache 的内存层，并在每个 SN 旁共置 Disk Server，实现内存-存储的本地化分层。

## 相关概念

- [[RDMA]]
- [[Shared-Storage Architecture]]
