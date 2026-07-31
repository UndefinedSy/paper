---
type: concept
aliases: [在线归一化, Streaming Softmax]
---

# Online Softmax

## 定义

一种可在分块或流式输入上精确计算 softmax 的归一化方法。每行维护当前最大值 $m$ 与稳定指数和 $\ell$，新块到来时重标定旧统计量并合并新统计量。

## 数学形式

对旧块统计 $(m_1,\ell_1)$ 和新块统计 $(m_2,\ell_2)$：

$$
m=\max(m_1,m_2),\qquad
\ell=e^{m_1-m}\ell_1+e^{m_2-m}\ell_2.
$$

该更新与对完整拼接向量一次性做数值稳定 softmax 等价。

## 核心要点

1. 保存每行 $O(1)$ 的状态，而不需保存完整 score row。
2. 是 tile-wise exact attention 的关键，因为 attention softmax 沿 key 维耦合。
3. 与 [[FlashAttention]] 的输出重标定结合，能逐块累积精确 $PV$。

## 代表工作

- [[FlashAttention]]：在 GPU SRAM tiles 中使用 online softmax。

## 相关概念

- [[Softmax]]
- [[Tiling]]
- [[Self-Attention]]
