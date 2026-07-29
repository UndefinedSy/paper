---
type: concept
aliases: [大推荐模型, LRM, Large Recommender Model]
---
# Large Recommendation Model
## 定义
使用 Transformer 等可扩展架构建模超长用户行为与多模态上下文、并探索推荐 scaling law 的大规模推荐模型。
## 核心要点
1. 训练依赖大范围用户序列扫描和持续模型迭代。
2. 在线推断要求低延迟读取用户序列与模型 KV Cache。
3. 多版本模型并行使训练—服务数据管理规模迅速增长。
## 代表工作
- [[TokaDB]]：针对 LRM 的用户行为序列与 KV Cache 设计统一存储引擎。
## 相关概念
- [[User Behavior Sequence]]
- [[KV Cache]]
