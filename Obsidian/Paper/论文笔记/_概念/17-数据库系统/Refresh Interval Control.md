---
type: concept
aliases: [刷新间隔控制]
---

# Refresh Interval Control

## 定义

根据近期增量维护成本和集群负载动态设置持续查询刷新周期的控制策略。

## 核心要点

1. 下界避免过度刷新和 thrashing。
2. 高 CPU/I/O 利用率时提高上界以减少竞争。
3. 滑动窗口平滑瞬时维护成本波动。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Incremental Processing Mode]]
- [[Incremental View Maintenance]]
