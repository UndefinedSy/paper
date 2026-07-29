---
type: concept
aliases: [自适应备份读, Hedged Read]
---
# Adaptive Backup Reads
## 定义
当主副本响应超过动态截止时间时，向另一副本发起并行读取、采用最先成功响应的尾延迟治理机制。
## 核心要点
1. 隐藏慢盘、慢副本和瞬时拥塞造成的长尾。
2. 静态截止时间可能形成备份读风暴，需按设备压力闭环调整。
3. 首个响应成功后应取消剩余请求，控制额外 I/O。
## 代表工作
- [[TokaDB]]：每 100 ms 根据压力信号调节 deadline，CPU 开销低于 1%。
## 相关概念
- [[KV Cache]]
- [[Hedged Request]]
