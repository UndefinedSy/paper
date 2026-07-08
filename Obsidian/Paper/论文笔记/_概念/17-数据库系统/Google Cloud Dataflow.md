---
type: concept
aliases: [Cloud Dataflow, Google Dataflow]
---

# Google Cloud Dataflow

## 定义

Google Cloud 的统一编程模型和服务，用于批处理和流处理数据管道，基于 Dataflow Model 论文。

## 核心要点

1. 统一批处理和流处理编程模型
2. 支持无序、大规模、无边界的流数据处理
3. [[Bigtable]] 的 CDC Beam Adapter 运行在 Dataflow 上
4. 离线访问功能允许 Dataflow worker 直接读取 SSTable
5. 平衡正确性、延迟和成本的实用方法

## 代表工作

- The Dataflow Model (VLDB 2015): 统一编程模型

## 相关概念

- [[Bigtable]]
- [[Change Data Capture]]
- [[MapReduce]]
