---
type: concept
aliases: [束搜索, Beam Decoding]
---

# Beam Search

## 定义

在自回归解码的每一步扩展当前候选序列，并只保留累计概率最高的 $k$ 条候选的近似搜索算法。

## 数学形式

$$\mathcal{B}_{t+1}=\operatorname{TopK}_{k}\{(s\oplus v,\log P(s)+\log P(v\mid s)):s\in\mathcal{B}_t,v\in V\}$$

## 核心要点

1. 用 beam width $k$ 控制质量与计算/内存开销的折中。
2. 多个候选经常共享一段生成历史，因此可以共享 [[KV Cache]] blocks。
3. [[PagedAttention]] 用块级共享与 [[Copy-on-Write]] 避免传统实现复制整段 KV cache。

## 代表工作

- [[PagedAttention]]：在束宽 6 的实验中相对 Orca(Oracle) 达到 2.3× 吞吐。

## 相关概念

- [[Autoregressive Generation]]
- [[KV Cache]]
- [[Copy-on-Write]]
