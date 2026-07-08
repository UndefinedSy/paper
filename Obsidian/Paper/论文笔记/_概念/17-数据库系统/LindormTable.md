---
type: concept
aliases: [LindormTable, Lindorm宽表引擎]
---

# LindormTable

## 定义

LindormTable 是 Lindorm 多模型数据库的宽表引擎，完全兼容 Apache HBase API 和数据模型，采用云原生分布式架构解耦存储和计算，实现从零到 PB 级的弹性扩展。

## 核心要点

1. 数据按行键字典序组织，支持高吞吐点查和范围扫描
2. 可实时同步数据到其他引擎（搜索、向量、列存等）
3. LindormVector 的所有索引数据持久化在 LindormTable 的三张宽表中
4. 提供批量 get 操作，高效支持 LindormVector 的重排阶段

## 代表工作

- [[LindormVector]]: 紧密集成的存储引擎

## 相关概念

- [[LindormDFS]]
- [[LindormVector]]
