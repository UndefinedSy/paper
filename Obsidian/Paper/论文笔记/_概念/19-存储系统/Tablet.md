---
type: concept
aliases: [Bigtable Tablet, 表格]
---

# Tablet

## 定义

Bigtable 中行的连续段，是数据托管（由 tablet server）、加载、卸载、移动和负载均衡的基本单位。

## 核心要点

1. 行空间按 row key 字典序划分为连续段
2. 托管/加载/卸载/移动/负载均衡的基本单位
3. 自动分裂和合并以避免热点
4. 数据存储在 [[Colossus]] 上而非本地磁盘，tablet 移动不迁移数据
5. 单表可有超过 1.6 quadrillion 行，单集群 250M+ QPS
6. 冷 tablet 可以更大（减少管理开销），热 tablet 更小（快速分裂和重均衡）
7. 每个 tablet 独立跟踪来自各 source 的复制位置

## 代表工作

- [[Bigtable]]: 数据托管和负载均衡的基本单位

## 相关概念

- [[Bigtable]]
- [[Tablet Server]]
- [[SSTable]]
- [[Locality Group]]
