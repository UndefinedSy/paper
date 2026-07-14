---
type: concept
aliases: [MetaServer, 元数据服务器]
---

# MetaServer

## 定义

[[ByteStore]] 分布式文件系统中的元数据管理节点，维护 WAL 文件状态、Blob 的 ChunkServer 地址映射（Chunk 描述符）以及 consolidated-length 等关键元数据，是代理读写路径中的核心协调组件。

## 核心要点

1. **存储内容**：
   - Blob 与 Group 的绑定关系（blob-group mapping）
   - Chunk 描述符：包含起始偏移、长度、磁盘 ID、ChunkServer 网络地址（含 EC 编码块的完整副本集）
   - WAL 文件关联状态（WAL file association）
   - consolidated-length：已确认写入存储层的数据长度
2. **读路径作用**：ProxyServer 查询 MetaServer 获取目标 Blob 的物理位置，实现 One-Hop 读中的元数据发现
3. **写路径作用**：记录 blob-group 绑定，协调写分配；后台 apply 完成后删除 WAL 文件记录
4. **一致性语义**：
   - 无 WAL 关联时：consolidated-length 最终且不回滚，读可绕过写节点
   - 有 WAL 关联时：返回长度 0，需走替代读路径保证数据一致性

## 相关概念

- [[ByteStore]]: MetaServer 所属的分布式文件系统
- [[ChunkServer]]: MetaServer 维护其物理地址信息
- [[WAL]]: MetaServer 追踪 WAL 文件关联状态
- [[TierTable]]: TierTable-Assisted 读中与 MetaServer 协作确定读路径
- [[CLAPS]]: 通过 MetaServer 实现 Proxy 路由和 One-Hop 读的元数据发现

## 代表工作

- [[CLAPS]]: CLAPS: A Load-Aware Proxy Resource Pooling System, 2026 SIGMOD
