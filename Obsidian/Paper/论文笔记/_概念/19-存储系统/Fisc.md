---
type: concept
aliases: [Fisc, FISC, Large-Scale Cloud-Native File System]
---

# Fisc

## 定义

阿里云设计的大规模云原生文件系统，采用存储感知分布式网关（SaDGW）和高性能网络栈优化代理转发路径。发表于 FAST'23，是云存储代理架构优化的代表工作。

## 核心要点

1. **SaDGW（Storage-aware Distributed Gateway）**：存储感知的分布式网关，利用共享内存优化代理内部转发路径
2. **局限性**：仍需代理参与读路径，存在两跳转发开销；仅改进转发实现，未消除代理层的参与
3. **黑名单机制**：提供粗粒度的代理不可用恢复机制，但忽略物理资源的空间关联性（如机架级集中）
4. **参考意义**：CLAPS 的 One-Hop 读机制进一步消除了 Fisc 仍保留的代理读转发开销

## 相关概念

- [[ProxyServer]]: Fisc 优化的代理层服务器
- [[CLAPS]]: 在 Fisc 基础上进一步消除代理读转发、实现有状态 Proxy 池化

## 代表工作

- Qiang Li et al., "FISC: A Large-Scale Cloud-Native-Oriented File System", FAST'23
