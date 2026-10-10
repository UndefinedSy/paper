---
type: concept
aliases: [FASTER KV]
---

# FASTER

## 定义

微软研究院提出的单机并发 KV 存储引擎，核心特性是支持原地更新（in-place update）的 hybrid log 结构，在保持高吞吐的同时允许热数据直接在内存中原地修改，冷数据则落入日志尾部等待回收。

## 核心要点

1. Hybrid log：日志的"热"端（接近尾部）允许原地更新，避免每次更新都生成新版本追加日志；"冷"端数据只读，通过日志截断/压缩回收空间。
2. 与 [[SOS]] 共享"至多一次 I/O"的设计精神（通过原地更新避免多版本查找开销），但 FASTER 面向单机极致吞吐优化，不涉及分布式强一致性。
3. 支持 epoch-based 无锁并发控制，允许多线程并发访问 hybrid log 结构。

## 代表工作

- FASTER 原始论文（Chandramouli et al., SIGMOD 2018）。
- [[SOS]]：相关工作中指出 FASTER 分享其"一次 I/O"设计理念但专注单机吞吐而非分布式强一致性，是设计理念上最接近的学术系统之一。

## 相关概念

- [[Hash Index]]
- [[LSM-tree]]
