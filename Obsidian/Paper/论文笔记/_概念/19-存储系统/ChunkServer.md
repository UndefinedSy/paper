---
type: concept
aliases: [ChunkServer, 数据块服务器]
---

# ChunkServer

## 定义

[[ByteStore]] 分布式文件系统中负责实际数据存储的节点，通过 RPC API 提供数据 Chunk 的读写接口，是 [[CLAPS]] One-Hop 读机制中客户端直接访问的最终目标。

## 核心要点

1. **存储内容**：存储 WAL 文件（写入 ByteStore 性能池）和数据 Chunk（容量池持久化数据）
2. **接口**：提供 RPC API 供 Client 直接访问，支持 One-Hop 直接读路径
3. **元数据**：Chunk 的物理位置信息（磁盘 ID、ChunkServer 网络地址、起始偏移、长度）由 MetaServer 维护
4. **副本/纠删码**：返回的 ChunkServer 地址集合包含完整副本集（含数据块和编码块用于纠删码数据），缺失副本对应位置为 null

## 故障处理

- **CLAPS 黑名单机制**：针对 ChunkServer 故障采用双层黑名单（节点级 + 磁盘级）
  - RPC 超时 → 仅对应磁盘加入磁盘级黑名单
  - 其他 RPC 错误 → 整个节点加入节点级黑名单

## 相关概念

- [[ByteStore]]: ChunkServer 所属的分布式文件系统
- [[MetaServer]]: 维护 ChunkServer 地址的元数据服务器
- [[WAL]]: ChunkServer 存储的预写日志文件
- [[CLAPS]]: 通过 One-Hop 读机制让 Client 直接访问 ChunkServer

## 代表工作

- [[CLAPS]]: CLAPS: A Load-Aware Proxy Resource Pooling System, 2026 SIGMOD
