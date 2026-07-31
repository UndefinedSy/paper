---
type: concept
aliases: [分布式死锁检测]
---
# Distributed Deadlock Detection
## 定义
在事务与锁分散于多个节点的系统中检测并打破死锁的机制：汇总各节点的 waits-for 图，在其并集中寻找环。
## 核心要点
1. 单节点的本地 waits-for 图无法发现跨节点的环，必须做全局汇总。
2. [[Aurora Limitless Database]] 的做法：客户配置一个超时，等待超时的事务触发怀疑；系统选一个 router 从各 shard 收集 waits-for 图并检测环。
3. 发现环后 Limitless **随机选一个事务**作为受害者中止——策略简单，缺少基于已完成工作量或优先级的代价评估。
4. 针对 DDL 的死锁预防：DDL 总是先在一个 distinguished 节点加锁，成功后再并行向其余节点加锁，失败则中止，以此规避加锁顺序不一致导致的死锁（代价是多一个 roundtrip）。
5. 替代思路是基于时间戳的预防策略（wait-die / wound-wait），对本来就有全局时间戳的系统尤为自然。
## 代表工作
- [[AuroraLimitless]]：超时触发 + waits-for 图并集 + 随机受害者。
## 相关概念
- [[Two-Phase Commit]]
- [[Sharding]]
- [[AuroraLimitless]]
