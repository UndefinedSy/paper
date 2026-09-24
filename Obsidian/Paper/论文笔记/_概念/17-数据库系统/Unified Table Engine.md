---
type: concept
aliases: [统一表引擎]
---

# Unified Table Engine

## 定义

ByteHouse 在统一 schema、元数据和事务可见性下同时承载结构化、文本与向量数据的表存储引擎。

## 核心要点

1. 逻辑上采用 document–chunk 两级模型。
2. 物理上以 stable/delta segment 兼顾扫描和更新。
3. ByteKV staging 与列式文件组成两阶段写入路径。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[MVCC]]
- [[Snapshot Isolation]]
- [[Sniffer File Format]]
