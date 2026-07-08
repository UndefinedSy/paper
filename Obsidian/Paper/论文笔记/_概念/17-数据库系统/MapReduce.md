---
type: concept
aliases: [MapReduce Framework]
---

# MapReduce

## 定义

由 Google 提出的简化大规模数据处理的编程模型，通过 map 和 reduce 两个阶段实现分布式并行计算。

## 核心要点

1. Map 阶段：将输入数据并行处理，输出中间 key-value 对
2. Reduce 阶段：按 key 聚合中间结果
3. [[Bigtable]] 早期使用 MapReduce 进行大规模并行计算
4. [[Bigtable]] 的物化视图使用 server-side MapReduce 风格流水线
5. [[Coprocessor]] 框架在 tablet server 上编排 MapReduce 流水线
6. 促进了 FlumeJava、Apache Beam、Apache Spark 等后续框架

## 代表工作

- MapReduce: simplified data processing on large clusters (OSDI 2004)
- [[Bigtable]]: 物化视图内部使用 MapReduce 风格流水线

## 相关概念

- [[Bigtable]]
- [[Coprocessor]]
- [[Google Cloud Dataflow]]
- [[Materialized Views]]
