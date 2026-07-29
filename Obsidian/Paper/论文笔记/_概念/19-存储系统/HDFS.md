---
type: concept
aliases: [Hadoop Distributed File System, Hadoop 分布式文件系统]
---
# HDFS
## 定义
面向大文件流式访问的分布式文件系统，通过分块、多副本和中心化元数据提供容错与横向扩展。
## 核心要点
1. DataNode 存储 block，NameNode 管理 namespace 与位置元数据。
2. 适合高吞吐批处理，不以低延迟随机访问为首要目标。
3. 常作为计算存储分离数据库的共享后端。
## 代表工作
- [[TokaDB]]：作为可插拔存储后端及 HBase、Doris 基线的底座。
## 相关概念
- [[Disaggregated Storage-Compute Architecture]]
- [[Shared-Storage Architecture]]
