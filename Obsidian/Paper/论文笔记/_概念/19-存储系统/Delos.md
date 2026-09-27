---
type: concept
aliases: [Delos, Virtual Consensus]
---
# Delos

## 定义
Delos 是 OSDI 2020 提出的**虚拟共识（virtual consensus）**系统：把共识协议的「日志复制」与「跨配置的状态机转移」解耦，使底层日志实现（Loglet）可以像虚拟化资源一样被替换与复用。

## 核心要点
1. **核心抽象是 Loglet**：一个 Loglet 只负责某个配置下的日志复制；Delos 通过在不同 Loglet 之间做透明的**所有权交接（fencing + 交接）**，把多个 Loglet 拼成一条逻辑上连续的日志。
2. **关键机制是 fencing**：交接时必须确保旧 Loglet 的写者不再写入，否则会产生分裂日志。这与 [[K2Plus]] 用 [[Seal]] 建立「唯一恢复边界」的思路同源。
3. **[[K2Plus]] 的引用方式**：论文把 Delos 与 Azure Append Blob 并列，作为「提供类似 sealing/fencing 原语」的存储系统例子，用来论证 K2+ 对存储层的要求（append-only 日志 + sealing）并非 DFV 独有，因而具备可移植性。
4. **价值**：说明「日志段的单写者边界」是分布式存储中反复出现的通用需求，而非某个产品的专有特性。

## 代表工作
- [[K2Plus]]：在可移植性论证中引用 Delos 的 fencing 原语作为同类机制。
- [[DFV]]：提供生产级的 Seal 原语。

## 相关概念
- [[Seal]]
- [[Plog]]
- [[Split-Brain]]
- [[DFV]]
