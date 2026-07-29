---
type: concept
aliases: [优先级资源管理, Priority-Aware Resource Management]
---
# Priority-Based Resource Management
## 定义
依据业务重要性在 CPU、缓存和 I/O 等共享资源上实施差异化调度、隔离与限流的方法。
## 核心要点
1. 优先级必须在不同资源层保持一致语义。
2. 高优任务需要低尾延迟，低优任务仍需避免永久饥饿。
3. 单层控制无法解决跨资源干扰。
## 代表工作
- [[TokaDB]]：组合抢占线程调度、分层缓存和双层限流保护 serving。
## 相关概念
- [[Preemptive Thread Scheduler]]
- [[Hierarchical Cache Allocator]]
- [[Multi-Layer Integrated Rate Limiter]]
