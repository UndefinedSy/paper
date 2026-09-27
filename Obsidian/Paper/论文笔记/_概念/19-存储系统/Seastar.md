---
type: concept
aliases: [Seastar 框架]
---
# Seastar

## 定义
Seastar 是一个高性能的**事件驱动 C++ 服务端应用框架**（由 ScyllaDB 主导开源），采用基于协程的异步编程模型，是 [[K2Plus]] 数据节点的底层运行时。

## 核心要点
1. **share-nothing 按核分片**：Seastar 按 CPU 核划分资源，每核拥有**独立内存**，避免跨核锁与缓存争用。[[K2Plus]] 沿用该模型把每个核绑定为一个 [[Data Shard]]。
2. **全异步无阻塞**：所有 I/O 异步化，配合协程风格编程，使 K2+ 与 [[DFV]] 的异步存储交互不会阻塞数据路径——这也是 DFV 相较本地文件**吞吐开销 < 2%** 的直接原因（论文明确将这一低开销归功于 Seastar 的异步调度框架）。
3. **代价**：核间内存不共享，导致 shard 级别的 resharding 昂贵；K2+ 因此引入 [[Physical Partition]] 作为可搬运的中间粒度。
4. **协程与竞态**：异步框架下同一事务的不同持久化请求可能由不同协程发起并被网络延迟，这正是 [[K2Plus]] 需要 [[Persist-in-Progress]] 来串行化竞争性状态转换的原因。

## 代表工作
- [[K2Plus]]：数据节点基于 Seastar 构建，核绑定 shard 的架构直接源于 Seastar 的资源模型。

## 相关概念
- [[Data Shard]]
- [[Physical Partition]]
- [[Persist-in-Progress]]
- [[DFV]]
