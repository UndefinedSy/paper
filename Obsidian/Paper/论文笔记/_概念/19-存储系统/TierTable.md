---
type: concept
aliases: [TierTable, 分层表, 写缓冲]
---

# TierTable

## 定义

[[ByteTier]] 中 RW ProxyServer 的内存写缓冲结构，在 Blob 写入期间缓存数据，同时作为直接路径读的加速组件，使 Client 可以直接从 ProxyServer 读取尚未写回存储层的热数据。

## 核心要点

1. **双重角色**：主要作为写缓冲（缓存 Blob 写入期间的数据），扩展后也作为读加速器（[[CLAPS]] 的 TierTable-Assisted Read）
2. **活跃数据位置**：活跃写入的数据位于 TierTable 的高地址区域，读取时从高到低串行访问
3. **生命周期策略**：CLAPS 主动将数据尽快下刷到存储层，不为读优化而延长 TierTable 存活时间，长期引导更多请求走 One-Hop 直接路径
4. **一致性处理**：读前通过 Open 过程从 MetaServer 获取 consolidated-length，无 WAL 关联时读可绕过写节点，有关联时需走替代路径

## 读加速效果

- 相比 Pure One-Hop：延迟最高减少 1996.21 μs（1G 文件配置）
- 20G 文件配置下减少 743.32 μs
- 在网络带宽受限场景下优势尤为明显

## 相关概念

- [[ByteTier]]: TierTable 所属的分层管理框架
- [[WAL]]: ByteTier 持久化前台写的日志文件，与 TierTable 协作
- [[CLAPS]]: 利用 TierTable 实现读加速的代理池化系统
- [[ChunkServer]]: TierTable 数据最终持久化的目标存储节点

## 代表工作

- [[CLAPS]]: CLAPS: A Load-Aware Proxy Resource Pooling System, 2026 SIGMOD
