---
type: concept
aliases: [PSD, Power Spectral Density, 功率谱密度]
---

# PSD (Power Spectral Density)

## 定义

功率谱密度是信号处理中描述信号功率在频率域分布的函数，通过傅里叶变换将时域信号转换为频域表示，用于识别信号中的周期性成分。

## 数学形式

$$S_{xx}(f) = \lim_{T \to \infty} \frac{1}{T} \left| \int_{-T/2}^{T/2} x(t) e^{-j2\pi ft} dt \right|^2$$

## 核心要点

1. PSD 峰值对应信号的主要周期，峰值所在频率 $f$ 对应周期 $T = 1/f$
2. ABase 用 PSD 分析时序的周期性，以确定使用 Prophet 还是历史均值等不同预测模型
3. 工程上常用 FFT 近似计算

## 代表工作

- Tobar et al., NIPS 2015: "Learning Stationary Time Series Using Gaussian Processes with Nonparametric Kernels"
- [[ABase]]: 集成预测方案的预处理步骤——用 PSD 判断时序周期性

## 相关概念

- [[Prophet]]
- [[变点检测|Change Point Detection]]
