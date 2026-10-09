---
type: concept
aliases: [ARIMA, AutoRegressive Integrated Moving Average]
---

# ARIMA

## 定义

自回归积分滑动平均模型（AutoRegressive Integrated Moving Average），一类对时间序列做差分平稳化后用自回归与滑动平均项建模的经典预测方法。

## 数学形式

$$
\left(1 - \sum_{i=1}^{p}\phi_i L^i\right)(1-L)^d y_t = \left(1 + \sum_{j=1}^{q}\theta_j L^j\right)\varepsilon_t
$$

## 核心要点

1. **三个阶数**：$p$ 为自回归阶数，$d$ 为差分阶数，$q$ 为滑动平均阶数，通常记作 ARIMA$(p,d,q)$。
2. **积分项 $(1-L)^d$**：通过差分把非平稳序列转为平稳序列，使模型可用于带趋势的指标。
3. **典型用途**：对具有周期性与趋势的业务指标（QPS、访问量、负载）做短期预测。
4. **在数据库系统中的角色**：作为负载预测器，为资源调度决策提供输入。

## 代表工作

- [[Nova]]: 用 ARIMA 分析索引的历史 QPS 来预测未来访问高峰，输出 [[Workload-Aware Pre-Warming|索引预热]]的预测分 $S_{\text{pred}}$。

## 相关概念

- [[Workload-Aware Pre-Warming]]
- [[Cold Start]]
