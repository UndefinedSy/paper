---
type: concept
aliases: [运行时过滤器]
---

# Runtime Filter

## 定义

查询执行期间从上游或 join build 端动态生成、并推入下游 scan 的过滤结构。

## 核心要点

1. 常见实现包括 Bloom filter、min-max 或值集合。
2. 提前减少 probe、网络传输和后续算子输入。
3. ByteHouse 还把跨表候选 id filter 推入 vector scan。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。
- [[TDSQL]]：提出 [[Adaptive Runtime Filter]] 分层自适应机制，实现 Bloom/Range/In 三类 filter 并解决广播开销与低选择率性能倒退问题。

## 相关概念

- [[Bloom Filter]]
- [[Hybrid Search]]
- [[Adaptive Runtime Filter]]
- [[Forward Node]]
