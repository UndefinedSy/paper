---
type: concept
aliases: [Sniffer 自描述文件格式]
---

# Sniffer File Format

## 定义

ByteHouse 将列数据、布局索引、统计、schema 与完整性信息共置于单文件的自描述列式格式。

## 核心要点

1. Data/Descriptor/Footer 三个 region 构成层次布局。
2. 点查通过 sort-key descriptor 和 layout index 精确定位 DataBlock。
3. 消除外置索引文件，降低 metadata lookup 与随机 I/O。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[Length-and-Presence Encoding]]
- [[Bloom Filter]]
- [[Apache Arrow]]
