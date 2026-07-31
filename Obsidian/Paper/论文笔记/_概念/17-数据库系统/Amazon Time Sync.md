---
type: concept
aliases: [Amazon Time Sync Service, Time Sync]
---
# Amazon Time Sync
## 定义
AWS 提供的高精度时间同步服务：在每个 region 用一组冗余的、卫星连接的原子钟为 EC2 实例提供**有界误差**的时间读数。
## 核心要点
1. API 返回当前时间 $t$ 与 **clock error bound (CEB)**，真实时间保证落在 $[t-\text{CEB},\ t+\text{CEB}]$ 内。
2. 典型 CEB 在所有 AWS region 都 **<1 毫秒**，部分 region 已降至**低双位数微秒**。
3. 该 API 由开源的 [[ClockBound]] daemon 暴露，所有 AWS 客户可用。
4. [[Aurora Limitless Database]] 用 `now().latest` 取快照与 prepare 时间戳，用 `now().earliest` 做 [[Commit Wait]] 与 [[Lease]] 判定。
5. ⚠️ 强依赖风险：整个正确性论证建立在 CEB 永不被违反的假设上，论文未讨论其失效或膨胀时的降级策略。
## 代表工作
- [[AuroraLimitless]]：全部时间戳与一致性保证的根基。
- It's about time: Microsecond-accurate clocks on Amazon EC2 instances (AWS, 2023)。
## 相关概念
- [[TrueTime]]
- [[ClockBound]]
- [[Commit Wait]]
- [[External Consistency]]
- [[Clock-SI]]
