---
type: concept
aliases: [脑裂]
---
# Split-Brain
## 定义
分布式系统的经典故障模式：因网络分区或误判故障，同时存在两个都认为自己有写权限的实例，导致数据分叉与不一致。
## 核心要点
1. 根因通常是「故障检测不可靠」——无法区分「节点挂了」与「网络不通」。
2. 常见解法：多数派（quorum）、fencing token、[[Lease]]，或依赖存储层的单写者语义。
3. [[Aurora Limitless Database]] 采用最后一种：[[Aurora Distributed Storage]] 对给定卷同一时刻只接受一个实例写入，故只有替代实例能写；原实例（zombie）在写失败后自行退出。
4. ⚠️ 单写者语义只解决**写**分叉，不解决 zombie 的**陈旧读**——后者仍需 [[Lease]] 兜底。
## 代表工作
- [[AuroraLimitless]]：存储层单写者 + 读租约的双重防护。
## 相关概念
- [[Lease]]
- [[Aurora Distributed Storage]]
- [[External Consistency]]
