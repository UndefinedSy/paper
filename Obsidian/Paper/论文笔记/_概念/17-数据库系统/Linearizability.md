---
type: concept
aliases: [线性一致性]
---

# Linearizability

## 定义

并发对象的最强一致性条件：每个操作看起来都在其调用和返回之间的某个瞬间原子地发生，且所有操作的这个"瞬间"构成一个与真实时间顺序兼容的全局顺序。

## 核心要点

1. 核心要求：如果操作 A 在操作 B 开始前已经完成，那么全局顺序中 A 必须排在 B 之前。
2. 实现线性一致性通常需要确定一个唯一的"线性化点"（linearization point），所有并发操作在逻辑上以该点的发生顺序排序。
3. 分布式系统中常见做法：所有读写都经过单一权威节点（如 primary/leader）、或通过共识协议（Paxos/Raft）确定全局顺序。
4. [[SOS]] 通过三个机制实现线性一致性：(1) 未提交版本对读严格不可见；(2) [[Lock Striping|锁条带]]序列化索引更新，定义线性化点为"head 应用 commit 并更新索引"的瞬间；(3) 路由表版本广播机制保证陈旧请求被拒绝，传播窗口小于因果间隔。

## 代表工作

- [[SOS]]：Primary-Based Chain Replication 中把线性化点定义为 head 索引更新的瞬间。
- [[CRAQ]]：允许多副本读，仍通过版本号机制保证线性一致性语义。

## 相关概念

- [[Chain Replication]]
- [[Two-Phase Commit]]
- [[Lock Striping]]
