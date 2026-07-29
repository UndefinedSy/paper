---
type: concept
aliases: [Hard Disk Drive, 机械硬盘]
---
# HDD
## 定义
使用旋转磁盘和机械磁头持久化数据的块设备，具有低容量成本和较高顺序带宽，但随机 IOPS 与尾延迟较弱。
## 核心要点
1. seek 与旋转延迟使随机读取昂贵。
2. 适合冷数据、大对象和顺序访问。
3. 慢盘与瞬时拥塞需要副本读、对齐和限流协同治理。
## 代表工作
- [[TokaDB]]：通过三项协同 I/O 技术在 HDD 上承载百 TB 级 KV Cache。
## 相关概念
- [[SSD]]
- [[Adaptive Backup Reads]]
- [[Hardware-Aware Block Alignment]]
