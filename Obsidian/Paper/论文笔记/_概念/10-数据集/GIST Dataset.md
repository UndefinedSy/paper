---
type: concept
aliases: [GIST-1M, GIST1M, GIST]
---

# GIST Dataset

## 定义

ANN benchmark 中常用的图像特征数据集：以 GIST 描述子表征图像的全局空间包络，广泛用于评测近似最近邻检索。

## 核心要点

1. **规模**：常用版本 GIST-1M 含 **100 万条 960 维**向量，通常按欧氏距离检索。
2. **来源**：GIST 描述子是 Oliva & Torralba 提出的场景「空间包络」整体表示，常见于图像检索研究。
3. **在向量数据库评测中的角色**：中等规模、高维度的经典基线，与 SIFT-1M 并列使用。
4. **特点**：维度高（960）使页式存储的内部碎片问题尤其突出——1024 维单精度向量已超 4KB。

## 代表工作

- [[Nova]]: 用于整体性能、索引构建时间、存储占用与 CIC 消融实验。

## 相关概念

- [[Cohere Dataset]]
- [[Tongyi Dataset]]
- [[VectorDBBench]]
- [[ANNS]]
