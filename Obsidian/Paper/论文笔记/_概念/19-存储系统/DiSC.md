---
type: concept
aliases: [DiSC, DISC, Distributed Shared Connection]
---

# DiSC

## 定义

通过多层 TCP 连接共享在多层应用中实现灵活旁路冗余处理层的系统，使用有状态内核连接（stateful kernel connections）绕过代理转发，减少 I/O 开销。发表于 NSDI'25。

## 核心要点

1. **核心思想**：多层 TCP 连接共享，使真实服务器可直接响应客户端，绕过负载均衡器/代理的下行开销
2. **局限性（CLAPS 视角）**：依赖全局同步序列（globally synchronized sequences），引入实现复杂度并限制系统灵活性
3. **背压缓解**：解决多层应用中的背压传播问题（Backpressure Mitigation）

## 相关概念

- [[ProxyServer]]: DiSC 旁路的代理层
- [[CLAPS]]: 选择 One-Hop 读 + Client Cache 方案，避免 DiSC 的全局同步复杂性

## 代表工作

- Brice Ekane et al., "DISC: Backpressure Mitigation In Multi-tier Applications With Distributed Shared Connection", NSDI'25
