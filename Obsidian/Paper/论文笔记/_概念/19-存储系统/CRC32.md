---
type: concept
aliases: [CRC-32, Cyclic Redundancy Check 32]
---

# CRC32

## 定义

32 位循环冗余校验码，用于检测数据传输或存储中的错误，是 Bigtable 数据完整性保护的主要工具。

## 核心要点

1. Bigtable 在多个数据粒度级别使用 CRC32 校验
2. 校验覆盖：row key、column key、value、mutation、文件级
3. 客户端可在写入时提供 CRC32C 校验和（或由客户端库自动生成）
4. 写路径多处验证：RPC 接收、commit log 写入前、memtable 更新前、复制路径
5. 读路径在 tablet server 和客户端两侧验证
6. 校验和持久化在 SSTable 中数据旁边
7. 压缩时验证校验和，错误则 tablet 隔离

## 代表工作

- [[Bigtable]]: 端到端校验的核心工具

## 相关概念

- [[Bigtable]]
- [[SSTable]]
- [[WAL]]
