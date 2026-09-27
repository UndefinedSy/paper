---
type: concept
aliases: [MVTO, 多版本时间戳排序]
---
# MVTO

## 定义
Multi-Version Timestamp Ordering，多版本时间戳排序：一种并发控制协议，为每个事务分配唯一时间戳，并借助多版本数据按时间戳顺序校验读写冲突，从而实现可串行化。

## 数学形式
事务 $T_i$ 的时间戳 $ts(T_i)$ 决定其串行化位置；读操作返回版本链中满足
$$ts(T_i) \ge w\text{-}ts(x_k) \quad \text{且} \quad ts(T_i) < w\text{-}ts(x_{k+1})$$
的版本 $x_k$，写操作需满足 $ts(T_i) \ge \max(r\text{-}ts(x))$。

## 核心要点
1. **无锁读**：读不加锁，通过多版本快照实现，天然适配只读事务与地理分布式读取。
2. **校验式写**：写需比对读时间戳；若事务的写会破坏既有读序，则该写被中止。
3. **时间戳来源是成败关键**：集中式授时会成为瓶颈，因此分布式实现常配 [[TrueTime]] 或 [[Hybrid Logical Clock]]。
4. [[K2]]/[[K2Plus]] 采用改造版 MVTO，用 [[Read Tracker]] 与 [[Write Stream]] 记录冲突信息而非完整版本链校验。

## 代表工作
- [[K2Plus]]：结合 [[TrueTime]] 的 MVTO，将 provisional 写先落 WS 立即持久化，再异步 finalize 为永久版本。
- [[Spanner]]：以 TrueTime 提供时间戳、以 Paxos 复制版本。

## 相关概念
- [[TrueTime]]
- [[Timestamp-Based MVCC]]
- [[Read Tracker]]
- [[Write Stream]]
- [[Serializability]]
