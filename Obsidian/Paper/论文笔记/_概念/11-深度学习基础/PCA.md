---
type: concept
aliases: [PCA, Principal Component Analysis, 主成分分析]
---

# PCA (Principal Component Analysis)

## 定义

PCA 是一种降维技术，通过正交变换将数据投影到方差最大的方向，保留前 $k$ 个主成分以最小化信息损失。

## 数学形式

$$
\mathbf{c}' = \mathbf{W}_{\text{PCA}}^T \mathbf{c}, \quad \text{retaining components explaining 95\% variance}
$$

## 核心要点

1. LindormVector 在质心搜索中用 PCA 压缩全精度质心
2. 保留解释 95% 方差的主成分（如 BioASQ 1024→384, Cohere 768→320）
3. 图遍历使用压缩质心减少内存带宽，再用全精度质心轻量重排
4. 这是 LindormVector 优化质心搜索的关键创新之一

## 代表工作

- [[LindormVector]]: PCA 压缩质心 + 重排优化 HNSW 遍历

## 相关概念

- [[HNSW]]
- [[LindormVector]]
