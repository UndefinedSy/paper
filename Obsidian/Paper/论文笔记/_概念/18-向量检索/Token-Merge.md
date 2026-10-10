---
type: concept
aliases: [Token-Merge, token-merge operation]
---

# Token-Merge

## 定义

一种针对多向量（token 级）表示的压缩技术，通过合并相似或冗余的 token 向量来降低存储与计算开销，常与量化方法配合使用。

## 核心要点

1. 用于压缩 [[ColBERT]] 等 token 级多向量表示的存储体积，使其接近稠密向量的存储量级。
2. 常与 [[Product Quantization|量化]]（如 Product Quantization）配合，共同实现"存储开销可比稠密向量、同时保留多向量检索精度"的目标。

## 代表工作

- [[VikingMem]]: 在预计算并存储记忆的 ColBERT 向量时，应用 token-merge 操作配合量化压缩存储开销。

## 相关概念

- [[ColBERT]]
- [[Product Quantization]]
