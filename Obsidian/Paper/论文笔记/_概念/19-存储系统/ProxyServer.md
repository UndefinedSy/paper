---
type: concept
aliases: [ProxyServer, 代理服务器, 存储代理]
---

# ProxyServer

## 定义

云存储系统中位于客户端与底层存储节点之间的代理层服务器，负责请求路由、元数据查询、数据转发和协调，屏蔽底层存储复杂性并提供统一访问接口。

## 核心要点

1. **功能**：请求路由、元数据查询（向 MetaServer）、写聚合（TierTable + WAL）、数据转发（读路径）
2. **类型（CLAPS 中）**：
   - **RW ProxyServer**：处理读写请求，持有 TierTable 缓存近期写数据
   - **RO ProxyServer**：仅处理读请求，轻量级，从 MetaServer 查元数据后重定向 Client
3. **有状态性**：ByteTier 的两阶段写要求 ProxyServer 维护跨请求的瞬态状态（TierTable、WAL 关联），这是 Proxy 池化的核心挑战
4. **Monitor 组件**：每个 ProxyServer 内置 Monitor 追踪流量和可用性指标，上报 ProxyMaster 用于集中管理
5. **处理粒度**：最细粒度服务单元为 **group**，每个 Blob 写请求在特定 group 中处理

## 传统架构问题

静态绑定（每个 ProxyServer 集群绑定一个 ByteStore 集群）导致：
- 需按各集群峰值独立预配资源
- 集群峰谷不同步 → 大量资源冗余
- 跨集群无法弹性共享

## 相关概念

- [[ByteTier]]: 引入有状态代理需求的两阶段写机制
- [[TierTable]]: RW ProxyServer 的写缓冲与读加速结构
- [[CLAPS]]: 将 ProxyServer 整合为共享弹性池的系统
- [[Proxy 资源池化]]: ProxyServer 弹性共享的核心设计范式

## 代表工作

- [[CLAPS]]: CLAPS: A Load-Aware Proxy Resource Pooling System, 2026 SIGMOD
- [[Fisc]]: 阿里云 FAST'23，SaDGW 存储感知分布式网关
