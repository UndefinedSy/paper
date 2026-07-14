---
type: concept
aliases: [ByteStore, 字节跳动分布式文件系统]
---

# ByteStore

## 定义

ByteDance 自研的面向大规模生产环境的分布式文件系统（DFS），采用 Blob 存储作为底层存储格式，支持追加写语义，面向高吞吐、读密集、成本敏感的多模态数据存储场景。

## 核心要点

1. **双池架构**：资源组织为两个独立可扩展的存储池——基于 SSD 的性能池（Performance Pool）和以 HDD 为主的容量池（Capacity Pool）
2. **Blob 存储模型**：以 Blob 为存储单元，支持追加语义，适合高吞吐写入场景
3. **双层接口**：对外提供 ChunkServer 接口（读写 WAL 文件和存储 Chunk）和 MetaServer 接口（维护 WAL 状态和 Chunk 信息）
4. **上层服务**：为 TOS（Torch Object Storage）、TLS（Torch Log Service）、RMQ（Rocket Message Queue）等服务提供存储基础

## 相关概念

- [[ByteTier]]: 基于 ByteStore 的两阶段写机制层
- [[ChunkServer]]: ByteStore 存储节点
- [[MetaServer]]: ByteStore 元数据服务器
- [[WAL]]: ByteStore 性能池中的预写日志文件
- [[CLAPS]]: ByteStore 上层的 Proxy 池化系统

## 代表工作

- [[CLAPS]]: ByteStore 和 ByteTier 之上的代理池化系统，2026 SIGMOD
