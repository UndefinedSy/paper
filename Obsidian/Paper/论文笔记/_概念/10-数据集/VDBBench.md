---
type: concept
aliases: [Vector Database Benchmark, VectorDBBench]
---

# VDBBench

## 定义

用于统一评估向量数据库吞吐、延迟与 recall 的 benchmark 框架。

## 核心要点

1. 通常固定 recall 比较 QPS。
2. 可组合 scalar filter 与向量检索。
3. **默认配置为单表场景**，因此多租户与跨表混合检索需要额外自建实验。
4. **常用指标**：[[Recall Amplification Factor|Recall]]、QPS（读）、TPS（写）与 mRT / P95 RT / P99 RT / MaxRT 延迟分位数。
5. **并发梯度**：常在 1、8、16、32、64、96 等并发级别下测量并汇报峰值 QPS。
6. **硬件口径**：论文常以 `ecs.g8i.4xlarge`（16 vCPU / 64 GiB）等规格对齐其默认配置。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。
- [[Nova]]：采用其默认单表配置做整体性能对比，并对多租户与跨表场景自建测试。

## 相关概念

- [[Milvus]]
- [[PGVector]]
- [[Hybrid Search]]
- [[ANNS]]
