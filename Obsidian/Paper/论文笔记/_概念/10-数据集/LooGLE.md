---
type: concept
aliases: [LooGLE, Long-Context Benchmark]
---

# LooGLE

## 定义

LooGLE 是一个用于评测长上下文语言模型理解能力的公开 benchmark，涵盖短依赖问答（shortdep_qa）、短依赖填空（shortdep_cloze）、长依赖问答（longdep_qa）等子任务。

## 核心要点

1. 覆盖不同 KV footprint 规模与依赖距离的长文本理解任务，适合评估 KV cache 系统在不同 context 长度下的复用效果。
2. shortdep_qa 的 KV footprint 相对较小，shortdep_cloze / longdep_qa 的 footprint 更大，更能暴露内存容量受限方案的瓶颈。

## 代表工作

- [[PolarKV]]：在 LooGLE 上与 [[Mooncake]]、[[3FS]] 对比 TTFT，footprint 增大时 PolarKV 的混合层级优势更明显。

## 相关概念

- [[SCBench]]
- [[KV Cache]]
