---
type: concept
aliases: [Amazon S3, S3]
---

# AWS S3

## 定义

Amazon Simple Storage Service，AWS 提供的高持久性、高可用的云对象存储服务，常被各类云原生数据库用作持久化系统记录（system of record）。

## 核心要点

1. 提供极高的数据持久性（通常标称 11 个 9），但随机读写尾延迟较高（可达秒级）
2. 常被数据库系统用作"最终真相来源"，而非直接服务低延迟请求——前端需叠加缓存层
3. 写一次语义（immutable object）与 LSM-tree 等不可变存储模型天然契合

## 代表工作

- [[SalesforceDB]]：[[SDBStore]] 以 S3 为云对象存储系统记录，叠加分布式缓存提供低延迟访问
- [[Socrates]]：同样将云对象存储作为持久层

## 相关概念

- [[SDBStore]]
- [[AWS EBS]]
- [[Immutability]]
