---
type: concept
aliases: [自适应运行时过滤器]
---

# Adaptive Runtime Filter

## 定义

[[TDSQL]] 提出的分层、自适应 [[Runtime Filter]] 机制，结合 group-wise cascading 通信策略与统计信息驱动的构建/绕过策略，解决传统 Runtime Filter 的广播开销与低选择率场景性能倒退问题。

## 核心要点

1. **通信层**：同机 Datanode 组成 group，[[Forward Node]] 先做组内聚合再做组间广播，把全连接广播转化为受控的分层交换。
2. **构建层**：基于 NDV 估计过滤有效性 $1-NDV_S(Z)/NDV_T(Z)$，低于阈值 $\beta$ 时直接禁用构建。
3. **执行层**：滑动窗口监控历史批次过滤率，动态决定当前/后续批次是否继续应用 filter，避免低选择率批次白白承担开销。

## 代表工作

- [[TDSQL]]：提出该机制以系统性缓解 Runtime Filter 的广播开销与静态脆弱性问题。

## 相关概念

- [[Runtime Filter]]
- [[Forward Node]]
- [[Bloom Filter]]
