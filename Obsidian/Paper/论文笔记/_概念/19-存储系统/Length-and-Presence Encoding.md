---
type: concept
aliases: [长度与存在性编码]
---

# Length-and-Presence Encoding

## 定义

为每个嵌套值或向量记录长度与存在性、使其成为独立物理单元的编码方式。

## 核心要点

1. 避免统一 offset/value 缓冲对稀疏和变长向量的浪费。
2. 支持 per-vector range、norm、nullness 等统计。
3. 连续向量切片利于块级压缩和 SIMD 解码。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Sniffer File Format]]
- [[Apache Arrow]]
