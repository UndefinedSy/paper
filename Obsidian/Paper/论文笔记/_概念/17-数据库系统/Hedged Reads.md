---
type: concept
aliases: [对冲读, Hedged Request]
---

# Hedged Reads

## 定义

一种降低尾延迟的技术：当一次读请求的首次尝试超过预期延迟阈值仍未返回时，向其他副本并发发出相同请求的"备份"，取最先返回的结果，以牺牲少量额外负载换取尾延迟的大幅降低。由 Jeffrey Dean 和 Luiz André Barroso 在《The Tail at Scale》中提出。

## 核心要点

1. 核心思想是用冗余请求对冲慢节点/慢路径带来的长尾延迟，而非单纯依赖单一路径的重试
2. 在触发阈值设置得当时，额外请求量可以很小（如仅对慢于 p95 的请求触发），但尾延迟改善显著
3. 常用于多副本存储系统的随机读路径

## 代表工作

- [[SalesforceDB]]：[[SDBStore]] 的数据缓存对每个 extent 维护 3 副本并跨 AZ 分布，使用 hedged reads 进一步降低随机读的尾延迟

## 相关概念

- [[SDBStore]]
- [[Bloom Filter]]
