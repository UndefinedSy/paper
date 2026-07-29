---
type: concept
aliases: [存算分离架构, Compute-Storage Disaggregation]
---
# Disaggregated Storage-Compute Architecture
## 定义
将无状态或弱状态计算节点与共享、独立扩展的持久存储层解耦，使计算能力和存储容量可以分别伸缩。
## 核心要点
1. 计算扩容不要求搬迁全部持久数据。
2. 共享存储提高资源池化和故障恢复能力。
3. 网络访问、远端 I/O 尾延迟和缓存一致性成为关键挑战。
## 代表工作
- [[TokaDB]]：Compute Node 通过统一 SDK 接入 ByteStore 或 HDFS。
- [[LindormVector]]：云原生多模型数据库采用存算分离架构。
## 相关概念
- [[Shared-Storage Architecture]]
- [[HDFS]]
- [[ByteStore]]
