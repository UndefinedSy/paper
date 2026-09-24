---
type: concept
aliases: [Microsoft MAchine Reading COmprehension]
---

# MS MARCO

## 定义

微软发布的大规模信息检索与机器阅读数据集，包含真实搜索查询与文档/段落。

## 核心要点

1. 常用于 passage ranking 和 dense retrieval。
2. ByteHouse 实验使用 8.84M passages 中抽样的 2M。
3. 配合 BGE-M3 评估文本、向量与混合召回。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[BGE-M3]]
- [[Hybrid Search]]
