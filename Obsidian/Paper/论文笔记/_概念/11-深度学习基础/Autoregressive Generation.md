---
type: concept
aliases: [自回归生成, Autoregressive Decoding]
---

# Autoregressive Generation

## 定义

按 token 顺序将序列联合概率分解为下一个 token 条件概率，并在每一步采样一个新 token 的生成范式。

## 数学形式

$$P(x_{1:n})=P(x_1)\prod_{i=2}^{n}P(x_i\mid x_{1:i-1})$$

## 核心要点

1. decode 的下一步依赖所有历史 token，无法跨时间步完全并行。
2. 保存历史 K/V 为 [[KV Cache]] 能避免每轮重复计算。
3. 长度未知使在线系统需要动态管理 KV 内存。

## 代表工作

- [[PagedAttention]]：针对自回归服务中动态增长的 KV cache 进行分页管理。

## 相关概念

- [[Causal Attention]]
- [[KV Cache]]
- [[Large Language Model Serving]]
