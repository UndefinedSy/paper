---
type: concept
aliases: [TT, Google TrueTime]
---
# TrueTime
## 定义
Google [[Spanner]] 使用的全局时间 API：由 GPS 与原子钟组成的时间基准，返回一个保证包含真实时间的区间而非单一时刻。
## 核心要点
1. 核心抽象是「有界不确定性」：`TT.now()` 返回 [earliest, latest]，真实时间必在其中。
2. 有了这个界，就可以用 [[Commit Wait]] 把时间戳顺序与真实时间顺序对齐，从而获得 [[External Consistency]]。
3. 它开创了「用硬件同步时钟为分布式事务定序」这一路线。
4. [[Amazon Time Sync]] 是 AWS 上的等效基础设施，误差界甚至更小（部分 region 已到低双位数微秒）。
## 代表工作
- [[Spanner]]：TrueTime 的宿主系统。
- Spanner, TrueTime and the CAP theorem (Brewer, 2017)。
## 相关概念
- [[Amazon Time Sync]]
- [[Commit Wait]]
- [[External Consistency]]
- [[Hybrid Logical Clock]]
