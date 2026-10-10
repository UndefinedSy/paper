---
type: concept
aliases: [SCBench, KV Cache-Centric Benchmark]
---

# SCBench

## 定义

SCBench 是一个以 KV cache 为中心的长上下文方法分析 benchmark，涵盖代码理解（Code.RepoQA）、英文多选（En.MultiChoice）、摘要（En.Sum）及多任务混合（Mix.Sum+NIAH、Mix.RepoQA+KV）等异构任务。

## 核心要点

1. 相比单一任务型 benchmark，SCBench 覆盖多种真实推理工作负载形态，更能反映生产环境下 KV cache 复用模式的多样性。
2. 任务设计强调 KV cache 的中心作用，便于横向比较不同 KV cache 系统（内存中心、存储中心、混合）在异构负载下的表现。

## 代表工作

- [[PolarKV]]：在 SCBench 代表性任务子集上评测，相对 [[3FS]] 降低 TTFT 47%–58%，相对 [[Mooncake]] 降低 3%–58%。

## 相关概念

- [[LooGLE]]
- [[KV Cache]]
