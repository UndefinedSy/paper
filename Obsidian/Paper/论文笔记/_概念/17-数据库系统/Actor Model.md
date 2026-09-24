---
type: concept
aliases: [Actor Model, Actor-Based Concurrency, Actor 模型, 行动者模型]
---

# Actor Model

## 定义

一种并发编程模型，将计算单元抽象为相互独立的「actor」，actor 之间仅通过消息传递通信，不共享可变内存，从而消除锁和共享状态带来的竞态条件。

## 核心要点

1. **消息传递代替共享内存**：每个 actor 拥有独立的内部状态和私有邮箱（消息队列），只能通过向其他 actor 发送消息来触发状态变更；不存在直接的内存共享。
2. **顺序处理消息**：单个 actor 顺序处理邮箱中的消息，内部逻辑天然无竞争；并发性来自多个 actor 的并行运行，而非单个 actor 内部的多线程。
3. **在 Firestore 实时栈中的应用**：早期使用共享内存 + 锁的架构导致 mutex 争用和悬垂指针类 bug 频发，改用基于请求队列的 actor 框架后，开发者必须显式表达并发行为，代码「更显然正确」，开发速度净提升。
4. **流式系统的天然适配**：实时栈的核心是「读写在何处相遇（rendezvous）」的流式数据处理，这种以消息流为中心的模式与 actor model 高度吻合；而传统请求-响应模型下锁的粒度难以设计正确。
5. **典型实现**：Erlang/Elixir OTP、Akka（JVM）、Pony；消息队列（如 Go channel）可视为轻量级 actor 思想的应用。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026) — 第 9 节 Lessons in Practice

## 相关概念

- [[ChangeLog]]
- [[QueryMatcher]]
- [[Realtime Query]]
