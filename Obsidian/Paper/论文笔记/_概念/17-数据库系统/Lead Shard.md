---
type: concept
aliases: [主导分片, 领导分片]
---
# Lead Shard
## 定义
[[Aurora Limitless Database]] 在 [[Two-Phase Commit]] 中指定的一个特殊参与分片：事务的 2PC 状态与最终 commit 时间戳持久化在它上面，而非持久化在协调者 router 上。
## 核心要点
1. 动机：router 为省成本没有专用 standby，故障后替换需数分钟，会长时间阻塞 2PC。
2. shard 可以配置 0～2 个 standby，因此 lead shard 的替换与状态恢复是秒级的。
3. 由 router 从「本事务更新过的 shard」中选出，其标识随 PREPARE 命令广播给其它参与者。
4. 它负责计算最终 commit 时间戳 = max(router 汇总的各 prepareTs, 自身时间戳提案)。
5. router 故障时它成为权威：其它 shard 向它查询事务状态；它已提交则全体提交，否则全体中止。
6. 读路径上它还承担「询问应答」职责：被问及某个 prepare 中事务时，或回复 commitTs，或抬高自身逻辑时钟以把该事务推到询问者快照之外。
## 代表工作
- [[AuroraLimitless]]：本概念的提出者，用它把协调者高可用问题转化为零成本的责任重分配。
## 相关概念
- [[Two-Phase Commit]]
- [[Hybrid Logical Clock]]
- [[External Consistency]]
- [[AuroraLimitless]]
