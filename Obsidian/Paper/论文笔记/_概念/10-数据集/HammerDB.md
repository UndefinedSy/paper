---
type: concept
aliases: [HammerDB benchmark]
---
# HammerDB
## 定义
业界标准的开源数据库基准测试工具，其 TPROC-C 负载派生自 [[TPC-C]]，以 NOPM（New Orders Per Minute）为主要吞吐指标。
## 核心要点
1. 支持 PostgreSQL、MySQL、Oracle、SQL Server 等多种数据库，负载由存储过程驱动。
2. 核心指标：**NOPM**（每分钟新订单数）与各存储过程（如 NEWORD）的延迟。
3. [[Aurora Limitless Database]] 使用**定制版**：修改建表脚本以支持分片表——8 张表按各自 `id` 分片，`item` 设为参考表，并把可单分片完成的函数标记为 distributed。
4. 论文实验参数：12000 warehouses、1000 虚拟用户、1000 万次迭代、每配置 1 小时（另有 20 分钟 ramp-up）、约 10% 分布式事务。
5. ⚠️ 「10% 分布式事务」是一个对系统友好的设置，论文未评估更高比例下的表现。
6. 论文的定制修改未公开，因此该实验不完全可复现。
## 代表工作
- [[AuroraLimitless]]：唯一使用的基准。
- HammerDB: The industry standard open-source database benchmark。
## 相关概念
- [[TPC-C]]
- [[Sharding]]
