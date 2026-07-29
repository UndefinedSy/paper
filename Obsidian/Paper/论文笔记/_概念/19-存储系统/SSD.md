---
type: concept
aliases: [Solid-State Drive, 固态硬盘]
---
# SSD
## 定义
使用闪存介质提供低延迟、高 IOPS 的持久存储设备，单位容量成本通常高于 HDD。
## 核心要点
1. 随机访问性能优于机械硬盘。
2. 写耐久、容量价格与写放大是系统设计约束。
3. 常与 HDD 组成冷热分层。
## 代表工作
- [[TokaDB]]：把近期用户行为 segment 保留在 SSD。
## 相关概念
- [[HDD]]
- [[Hot-Cold Tiering]]
