---
type: concept
aliases: [PrestoDB, Presto]
---

# PrestoDB

## 定义

采用计算存储解耦架构的分布式 SQL 查询引擎，常搭配 Apache ORC 文件格式与 HDFS 等外部存储层。

## 核心要点

1. 引入 TCP 连接多路复用或按服务器连接池缓解 socket 耗尽问题，但未解决拓扑层面的低效根源。
2. 解耦存储带来弹性伸缩优势，但磁盘到内存的数据加载增加了 I/O 开销，导致查询延迟高于紧耦合架构。
3. 部分实现了向量化改进，但相比 [[DuckDB]]、[[TDSQL]] 仍不完整。

## 代表工作

- [[TDSQL]]：作为计算存储解耦架构的对比基线。

## 相关概念

- [[MPP Architecture]]
- [[Vectorized Execution Engine]]
