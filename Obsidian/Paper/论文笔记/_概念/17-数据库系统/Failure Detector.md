---
type: concept
aliases: [最终完美故障检测器, Eventually Perfect Failure Detector, Unreliable Failure Detector]
---

# Failure Detector

## 定义

分布式系统中用于判断节点是否失效的抽象组件。Chandra & Toueg (1996) 定义了一系列不可靠故障检测器类别，按"完整性"（correctly detect crashed nodes）与"准确性"（not falsely suspect live nodes）的强弱组合分类；"最终完美"（Eventually Perfect, ◇P）故障检测器保证：最终所有崩溃节点都会被永久怀疑，且最终所有存活节点都不再被怀疑。

## 核心要点

1. 分布式系统中无法构建完全准确、及时的故障检测器（受网络延迟不确定性所限），因此转向"最终"保证：允许短暂的误判，但长期行为收敛到正确判断。
2. 常见实现：心跳机制——节点周期性发送心跳，监控方若超过超时阈值未收到心跳则怀疑该节点失效。
3. 超时阈值的选择在"快速检测故障"与"避免误判瞬时抖动"之间权衡。
4. [[SOS]] 的 ConfigServer 通过心跳机制维护每个 DataServer 的最后心跳时间，超时即判定失效并发起剔除，是 ◇P 故障检测器的典型工程实现。

## 代表工作

- [[SOS]]：ConfigServer 与 DataServer 的心跳机制，实现最终完美故障检测器语义，用于触发节点永久剔除。

## 相关概念

- [[Chain Replication]]
- [[Two-Phase Commit]]
