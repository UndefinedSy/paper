---
type: concept
aliases: [C++ Workflow, Sogou Workflow]
---

# Workflow

## 定义

搜狗（Sogou）开源的 C++ 异步服务端框架，提供统一的异步任务抽象（network、computation、timer、file I/O 等），将网络 I/O 与计算彻底解耦，支持高并发场景下充分利用 CPU、网络、磁盘资源。

## 核心要点

1. 核心抽象是"任务"（task）与"任务图"（series/graph），把异步操作组织为可编排的流水线，而非传统回调地狱式的异步编程模型。
2. 通过事件驱动 + 线程池结合的方式避免阻塞式 I/O，并避免软件层调度自身成为系统瓶颈。
3. 作为通用基础设施框架，被多个腾讯内部系统复用（包括 [[SOS]] 的 DataServer 间通信层）。

## 代表工作

- [[SOS]]：DataServer 间通信基于 C++ Workflow 异步框架实现，分离网络 I/O 与计算，支撑链复制协议下的高并发请求处理。

## 相关概念

- [[Chain Replication]]
