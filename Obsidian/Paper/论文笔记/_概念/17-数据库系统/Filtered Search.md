---
type: concept
aliases: [Filtered Search, 过滤检索]
---

# Filtered Search

## 定义

过滤检索将向量搜索与结构化属性谓词（如类别、时间戳、用户 ID）结合，先估计谓词选择性再动态选择策略。

## 核心要点

1. 预过滤（高选择性）：先评估谓词，再对合格行做向量比较
2. 后过滤（低选择性）：先向量搜索，再在协调器评估谓词；不足时增大 nprobe 重试
3. 属性下推（Attribute Pushdown）：结构化属性嵌入向量索引列式文件，扫描时先过滤再距离计算
4. LindormVector 的属性下推在全选择性范围保持约 10-13ms 延迟和 95% 召回率

## 代表工作

- [[LindormVector]]: 三策略动态选择 + 属性下推优化

## 相关概念

- [[Hybrid Search]]
- [[LindormVector]]
