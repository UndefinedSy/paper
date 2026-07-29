---
type: concept
aliases: [抢占式线程调度器, 用户态抢占调度]
---
# Preemptive Thread Scheduler
## 定义
允许高优先级任务中断低优先级任务、优先获得 CPU 的调度器；用户态实现可避免内核切换开销并提供数据库语义感知。
## 核心要点
1. 高低优队列显式区分 serving、training 与后台任务。
2. 异步 I/O 完成后的高优任务应立即恢复执行。
3. 需要预留少量 CPU 给低优任务，避免永久饥饿。
## 代表工作
- [[TokaDB]]：每核维护 HPQ/LPQ，以亚毫秒级切换保护在线请求 P99。
## 相关概念
- [[Priority-Based Resource Management]]
