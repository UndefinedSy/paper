---
type: concept
aliases: [Prophet, Facebook Prophet, 时序预测Prophet]
---

# Prophet

## 定义

Facebook（Meta）开源的时序预测模型，基于可分解的加法模型（趋势 + 季节性 + 节假日效应），擅长处理有明显趋势和多重周期性的时序数据，无需大量调参即可在生产环境中使用。

## 数学形式

$$y(t) = g(t) + s(t) + h(t) + \epsilon_t$$

其中 $g(t)$ 为趋势项，$s(t)$ 为季节性，$h(t)$ 为节假日效应，$\epsilon_t$ 为噪声。

## 核心要点

1. 趋势项支持线性和逻辑增长两种形式，可自动检测 changepoint
2. 季节性通过 Fourier 级数建模，支持年/周/日多周期
3. 对缺失值和异常值有较好鲁棒性
4. ABase 中与历史均值集成使用：Prophet 处理有趋势/周期的序列；历史均值补充处理平稳序列

## 代表工作

- Taylor & Letham, 2018: "Forecasting at Scale", The American Statistician
- [[ABase]]: 集成预测方案中的 Prophet 组件

## 相关概念

- [[变点检测|Change Point Detection]]
- [[PSD|Power Spectral Density]]
