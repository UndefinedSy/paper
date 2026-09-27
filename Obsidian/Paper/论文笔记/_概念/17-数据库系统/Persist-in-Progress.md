---
type: concept
aliases: [PIP, 持久化进行中]
---
# Persist-in-Progress

## 定义
Persist-in-Progress（PIP）是 [[K2Plus]] 引入的事务/写状态：表示**系统已发出持久化请求但结果未知**。在不确定性消除之前，必须约束与之冲突的状态转换，从而不依赖 OS 级锁地串行化竞争性的持久化尝试。

## 核心要点
1. **它解决的真实竞态**：事务 $txn_1$ 的提交请求在网络中延迟；同时另一协程因心跳超时把它中止并成功写入 WAL；随后延迟的提交请求恢复，把状态作为 committed 持久化进 append-only log——于是「已被中止的事务最新状态却是 committed」。
2. **两类状态机**：事务状态机（`InProgress → CommitPIP → Committed → FinalizedPIP → End`）与写操作状态机（`InProgressPIP → InProgress → Committed/FinalizedPIP`）。
3. **重试策略按持久化点区分**：
   - **正确性攸关点**（provisional 写、提交状态）：`CommitPIP` 必须重试直到收到明确响应（成功**或**失败），不接受未知结果。
   - **可恢复点**（finalize 写、finalized 状态）：只做**有限次重试**，仍不明确则安全继续，把问题留给后续读写冲突解析阶段。
4. **全局保证**：每个事务只有一个提交或中止决定生效；延迟或重试的持久化请求不能覆盖已 finalized 的决定。

## 代表工作
- [[K2Plus]]：在 [[DFV]] 的 append-only [[Plog]] 上处理异步网络导致的 [[False-Positive Write|假阳性持久化失败]]。

## 相关概念
- [[K2Plus]]
- [[Write Stream]]
- [[WAL]]
- [[False-Positive Write]]
