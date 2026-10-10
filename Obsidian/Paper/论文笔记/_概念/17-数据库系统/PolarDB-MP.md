---
type: concept
aliases: [PolarDB-MP, PolarDB Multi-Primary]
---

# PolarDB-MP

## 定义

PolarDB-MP 是阿里云通过 disaggregated shared memory 实现的多主（multi-primary）云原生数据库，是 [[Disaggregated Memory Pool]] 技术的早期落地场景之一。

## 核心要点

1. 让多个计算节点（primary）共享同一份内存态数据，支持多主写入与一致性协调。
2. 其底层的分布式共享内存设计（Home Node + Slab Node 组织方式）被后续系统（如 PolarKV 的 KV cache 内存层）复用。

## 代表工作

- [[PolarKV]]：直接复用 PolarDB-MP 一脉的分布式内存池技术作为 KV cache 的内存层基础设施。

## 相关概念

- [[Disaggregated Memory Pool]]
- [[RDMA]]
