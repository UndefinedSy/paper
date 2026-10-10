---
type: concept
aliases: [CRAQ, Chain Replication with Apportioned Queries]
---

# CRAQ

## 定义

Chain Replication with Apportioned Queries：经典[[Chain Replication|链复制]]的变体，允许从链上任意副本（而非只能从 tail）读取数据以扩展读吞吐，用版本号区分"干净"读与"脏"读。

## 核心要点

1. 每个副本为同一 key 维护多个版本（已提交版本 + 可能存在的未提交 dirty 版本）。
2. 读请求命中非 tail 副本时，若该副本持有 dirty 版本，需要向 tail 查询最新已提交的版本号来判断是否可以直接返回本地版本（干净读）或需要额外一次网络往返确认（脏读路径）。
3. 相比经典链复制固定查 tail，CRAQ 用空间（存储多版本）换读吞吐扩展性，但在写争用激烈时脏读检测会增加延迟。
4. [[SOS]] 的设计选择与 CRAQ 相反：固定查 head（而不是允许任意副本），通过避免多副本读来避免脏读检测开销，用"一次 I/O，单跳读"换取可预测的低尾延迟。

## 代表工作

- [[DeepSeek 3FS]]：面向大规模顺序读写训练场景，采用 CRAQ 作为复制协议以获得更高读吞吐。
- [[SOS]]：相关工作部分明确对比 CRAQ，指出 SOS 为降低单请求尾延迟而选择 head-only 读路径，牺牲了 CRAQ 式的多副本读扇出能力。

## 相关概念

- [[Chain Replication]]
- [[Linearizability]]
