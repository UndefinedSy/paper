---
type: concept
aliases: [链复制]
---

# Chain Replication

## 定义

一种支持强一致性的副本协议：$R$ 个副本组成一条线性链，写请求从链头（head）进入、沿链传播到链尾（tail）才算提交；读请求查询链尾以保证线性一致性。

## 核心要点

1. 写路径：head 持久化后转发给下一个 follower，直到 tail 持久化完成，写才算成功。
2. 读路径：经典设计中读查询 tail，因为 tail 持有已确认提交到全部副本的最新数据。
3. 相比主从异步复制，链复制通过"全部确认才提交"获得强一致性，但增加写延迟（每多一个副本增加一跳）。
4. 变体可修改读路径（如从 head 读）或引入多阶段提交以优化可靠性和失败处理，典型例子见 [[CRAQ]]、SOS。

## 代表工作

- [[SOS]]：Primary-Based Chain Replication with Two-Phase Commit——把读路径改为查 head 而非 tail，并把链上提交拆分为 Reserve/Commit 两阶段，降低提交阶段失败概率。
- [[CRAQ]]：允许从任意副本读（而非只读 tail），用版本号区分脏读/干净读，提升读吞吐。

## 相关概念

- [[Two-Phase Commit]]
- [[Linearizability]]
- [[CRAQ]]
