---
type: concept
aliases: [Clock Snapshot Isolation]
---
# Clock-SI
## 定义
SRDS 2013 提出的方案：在分区数据存储上仅用松散同步的物理时钟即可实现 [[Snapshot Isolation]]，无需中心化的时间戳分配器。
## 核心要点
1. 快照与提交时间戳都取自各节点本地物理时钟，摆脱中心时间戳权威这一瓶颈与单点。
2. 核心难题是时钟偏移：读事务的快照时间戳可能领先于被读分区的本地时间。
3. Clock-SI 的解法是**延迟读**，直到该分区的本地时钟推进过读者的快照时间戳。
4. 另一个难题是读到已 prepare 未定论的事务，需要等待其定论。
5. [[Aurora Limitless Database]] 沿用其整体思路，但用 [[Hybrid Logical Clock]] 把「延迟读」替换为「抬高分区逻辑时钟」，从而免去读路径等待。
## 代表工作
- [[AuroraLimitless]]：Clock-SI 思路的工业级实现与改进。
## 相关概念
- [[Snapshot Isolation]]
- [[Hybrid Logical Clock]]
- [[Amazon Time Sync]]
- [[Timestamp-Based MVCC]]
