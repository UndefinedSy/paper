---
type: concept
aliases: [服务等级目标, Service Level Objective]
---
# SLO

## 定义
Service Level Objective（SLO）是对服务质量的可量化目标，例如可用性比例、延迟分位或错误率上限。它是 SLA 的内部实现目标。

## 核心要点
1. **可用性口径**：[[K2Plus]] 沿用 Google Cloud 的口径——**5 分钟窗口内服务器错误率 > 5%**（且请求量 ≥ 60 req/min）即计为一次 downtime。
2. **驱动设计取舍**：K2Plus 的「active-on-demand + 关键分区 pin 为 always-active」正是按 SLO 分级：普通分区容忍 [[Cold Start]]（约 27 s），SLO 攸关分区则牺牲资源占用换取零冷启动。
3. **复制模式即 SLO 分级**：同步复制（RPO≈0、吞吐成本 42%）服务于对一致性/持久性 SLO 最敏感的业务；异步复制服务于吞吐优先的业务。
4. **监控落地**：K2+ 对每个 VM 跟踪 CPU/内存/连接/异常进程，对服务器间请求跟踪延迟与失败率；超 5% 请求超时或出错持续两个统计周期即告警并隔离组件。

## 代表工作
- [[K2Plus]]：以 SLO 口径定义可用性，并据此设计分区温备策略与复制模式分级。

## 相关概念
- [[RPO]]
- [[Cold Start]]
- [[Hot Backup]]
- [[Load Balancing]]
