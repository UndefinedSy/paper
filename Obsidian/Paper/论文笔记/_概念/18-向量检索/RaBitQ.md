---
type: concept
aliases: [Binary Quantization, RaBitQ]
---

# RaBitQ

## 定义

RaBitQ 是一种二值量化方法，将 $d$ 维向量映射到 $d$ 维超立方体的顶点。先施加固定随机正交旋转，再取每个旋转后坐标的符号，生成 $d$-bit 二值码。

## 数学形式

$$
\text{RaBitQ}(\mathbf{v}) = \text{sign}(\mathbf{R} \cdot \mathbf{v})
$$

## 核心要点

1. 极致压缩：每个向量仅用 $d$ bits（1 bit per dimension）
2. 理论误差界：提供量化误差的理论上界保证
3. 随机正交旋转使各维度统计独立，编码后信息损失最小化
4. 可与 IVF 组合使用

## 代表工作

- [[LindormVector]]: 提供 PQ 和 RaBitQ 两种量化选项
- Gao & Long (2024): RaBitQ 原始论文

## 相关概念

- [[Product Quantization]]
- [[IVFPQ]]
