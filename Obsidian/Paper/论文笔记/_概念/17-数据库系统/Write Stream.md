---
type: concept
aliases: [WS, 写流]
---
# Write Stream

## 定义
Write Stream（WS）是 [[MVTO]] 类协议中的在途写记录：保存**所有尚未最终确定（finalize）的写及其创建者事务的时间戳**，用于阻塞那些会绕过未提交写的读。

## 核心要点
1. **provisional 条目**：每次成功的写先在 WS 中创建 provisional 条目并**立即持久化**，以防节点故障丢写。
2. **读阻塞**：绕过未提交写的读会被阻塞，直到该写提交或中止，从而保证不会读到「不存在」的中间状态。
3. **异步 finalize 的基础**：正因为 WS 条目 + 事务最终结果可以完全重建一个写的最终状态，[[K2Plus]] 才能把 finalize（把 provisional 条目转为永久版本）放到向客户端应答**之后**异步执行。
4. 与 [[Read Tracker]] 配对：WS 管未提交可见性，RT 管读序。WS 同样不持久化，因此节点故障时在途事务不被恢复。

## 代表工作
- [[K2Plus]]：WS 的四个持久化点之一是正确性攸关的（provisional 写必须可恢复），据此设计 [[Persist-in-Progress]] 状态机。

## 相关概念
- [[MVTO]]
- [[Read Tracker]]
- [[Persist-in-Progress]]
- [[WAL]]
