---
type: concept
aliases: [键值缓存, Attention KV Cache]
---

# KV Cache

## 定义

Transformer 推理中缓存各层历史 token 的 Key、Value 矩阵，以避免自回归生成时重复计算历史注意力状态。

## 核心要点

1. 用存储容量换取推理计算量与时延下降。
2. 在大模型 / 大推荐模型中，单条 cache 可达 MB 级，模型版本并行会形成超大数据集。
3. 在线 serving 需要低尾延迟随机读，训练过程则持续产生高带宽写入。
4. 大 value 对块对齐、写放大和 HDD 随机 I/O 极为敏感。
5. 在在线 [[Large Language Model Serving|LLM 服务]]中，KV cache 随 decode 动态增长，GPU 端的碎片、预留和重复副本会直接限制 batch size；[[PagedAttention]] 以定长块实现按需分配与共享。

## 代表工作

- [[TokaDB]]：将 KV Cache 作为与用户行为序列并列的核心数据域，用协同 I/O 引擎在 HDD 上兼顾读尾延迟和写带宽。
- [[ABase]]：生产负载中也包含远程 LLM KV Cache。

## 相关概念

- [[Hardware-Aware Block Alignment]]
- [[Adaptive Backup Reads]]
- [[Log as Data]]
