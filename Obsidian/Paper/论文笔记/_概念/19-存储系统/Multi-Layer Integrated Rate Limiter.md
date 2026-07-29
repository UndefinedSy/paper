---
type: concept
aliases: [多层集成限流器, 请求-硬件双层限流]
---
# Multi-Layer Integrated Rate Limiter
## 定义
同时在逻辑请求层和物理 I/O 层执行优先级感知限流，以控制请求入口并约束真实设备压力。
## 核心要点
1. 请求层 token 表达 serving 高于 training/background 的业务优先级。
2. 硬件层 token 约束实际 I/O，补偿同类请求 I/O 量不同的问题。
3. Sentinel 根据设备利用率和队列深度对低优流量施加 backpressure。
## 代表工作
- [[TokaDB]]：双层 throttler 避免训练突发抢占在线读带宽。
## 相关概念
- [[Priority-Based Resource Management]]
