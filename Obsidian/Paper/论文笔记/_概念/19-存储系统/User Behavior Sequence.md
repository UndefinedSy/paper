---
type: concept
aliases: [用户行为序列, User Interaction Sequence]
---
# User Behavior Sequence
## 定义
按时间排序记录用户点击、点赞、转发、购买等交互的长序列，是推荐模型学习长期兴趣和在线推断的重要输入。
## 核心要点
1. 在线 serving 常按用户读取最近若干条记录。
2. 离线 training 常跨大量用户和时间窗口做范围扫描。
3. 数据可达 PB 级，单用户序列可包含数十万条记录。
## 代表工作
- [[TokaDB]]：通过 Detached-Segment Index Layout 统一服务点读与范围扫描。
## 相关概念
- [[Large Recommendation Model]]
- [[Detached-Segment Index Layout]]
