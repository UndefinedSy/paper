---
type: concept
aliases: [租约, Read Lease, 读租约]
---
# Lease
## 定义
分布式系统中带时限的授权：持有者在租约有效期内可以行使某项权利（如提供读服务），到期未续约即自动失效，从而无需额外通信即可安全地判定权限。
## 核心要点
1. 核心价值：把「需要协调确认」的问题转化为「等一段时间即可确定」的问题。
2. [[Aurora Limitless Database]] 的用法：每个 shard 通过**成功写入存储**建立与续约租约；写入打上当前时钟值 $t$，成功后获得到 $t+\text{TTL}$ 为止的租约。
3. 该租约允许 shard 服务**快照时间 $\le t+\text{TTL}$** 的读；多次续约失败则 shard 断定自己是 zombie 并**自我终止**。
4. Failover 后新实例先从卷中恢复 $t_{\text{last}}$，等到 $\texttt{now().latest} > t_{\text{last}}+\text{TTL}$ 才开始处理事务。
5. 正确性关键：等待结束后新实例提交的事务 commitTs 必然 $> t_{\text{last}}+\text{TTL}$，故 zombie 能服务的读只可能漏掉本就不该看到的事务——[[External Consistency]] 得以保持。
6. 实践取值：TTL 设得**小于典型 failover 时间**，因此多数情况下无需额外等待。
## 代表工作
- [[AuroraLimitless]]：用写入驱动的租约防止 zombie shard 提供陈旧读。
- [[Chubby]]：以租约为核心原语的分布式锁服务。
## 相关概念
- [[Split-Brain]]
- [[External Consistency]]
- [[Amazon Time Sync]]
- [[Aurora Distributed Storage]]
