---
type: concept
aliases: [ClockBound daemon]
---
# ClockBound
## 定义
AWS 开源的守护进程，向应用暴露 [[Amazon Time Sync]] 的「时间 + 误差界」API。
## 核心要点
1. 提供 `now()` 返回 (t, CEB)，应用据此得到 earliest/latest 区间。
2. 把「时钟误差可量化」这一能力从内部基础设施变成所有 AWS 客户可用的公共接口。
3. [[Aurora Limitless Database]] 的时间戳获取即经由此 API。
4. 开源地址：github.com/aws/clock-bound。
## 代表工作
- [[AuroraLimitless]]：通过 ClockBound 获取有界时间。
## 相关概念
- [[Amazon Time Sync]]
- [[TrueTime]]
- [[Commit Wait]]
