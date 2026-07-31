---
type: concept
aliases: [Aurora, AWS Aurora]
---
# Amazon Aurora
## 定义
AWS 的云原生关系数据库服务，核心创新是把日志与存储下推到一个跨 3 AZ 的分布式存储层，使计算节点只需写 redo 日志。
## 核心要点
1. 「日志即数据库」：计算层只向存储层发 redo 日志，页面物化由存储节点后台完成，极大压缩了写放大与网络流量。
2. quorum 复制：每个 segment 复制到 6 个存储节点（3 AZ × 2），写 quorum 4/6、读 quorum 3/6。
3. 由此可容忍「整个 AZ 故障 + 另一 AZ 中一个副本故障」而不丢数据，整 AZ 故障下仍可写。
4. 避免为 I/O、提交与成员变更使用分布式共识（这是第二篇论文的核心主张）——[[Aurora Limitless Database]] 正是复用了这一点，从而无需 Paxos/Raft。
5. 关键衍生能力：[[Copy-on-Write Cloning]]（卷克隆）、单写者语义（用于规避 [[Split-Brain]]）、[[Aurora Serverless V2]]（按 ACU 弹性调容）。
## 代表工作
- Amazon Aurora: Design considerations for high throughput cloud-native relational databases (SIGMOD 2017)。
- Amazon Aurora: On avoiding distributed consensus for I/Os, commits, and membership changes (SIGMOD 2018)。
- [[AuroraLimitless]]：在其之上构建水平分片层。
## 相关概念
- [[Aurora Distributed Storage]]
- [[Aurora Serverless V2]]
- [[Copy-on-Write Cloning]]
- [[Log as Data]]
- [[Disaggregated Storage-Compute Architecture]]
- [[Split-Brain]]
