---
type: concept
aliases: [LindormDFS, Lindorm Distributed File System]
---

# LindormDFS

## 定义

LindormDFS 是 Lindorm 的云原生、可扩展、容错分布式存储层，被所有 Lindorm 引擎共享。基于 Apache HDFS 扩展，结合对象存储和块存储的优势。

## 核心要点

1. 完全兼容 Apache HDFS 协议
2. 文件分块为固定大小 block，跨节点复制保证持久性和高可用
3. 云原生能力：冷热分层存储、智能数据放置、共享副本去重
4. 存储与计算分离，独立扩展存储，避免资源耦合
5. LindormVector 的所有数据（索引 + 原始向量）持久化在 LindormDFS

## 代表工作

- [[LindormVector]]: 底层存储基础设施
- [[LindormTable]]: 宽表引擎的持久化层

## 相关概念

- [[LindormTable]]
- [[LindormVector]]
- [[Shared-Storage Architecture]]
