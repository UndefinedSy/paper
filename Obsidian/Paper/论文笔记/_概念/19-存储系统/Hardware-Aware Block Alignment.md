---
type: concept
aliases: [硬件感知块对齐, 物理块对齐]
---
# Hardware-Aware Block Alignment
## 定义
让逻辑对象边界与底层物理 I/O block 边界对齐，以避免一次逻辑读取跨越多个不连续物理块。
## 数学形式
$$PaddingSize=BlockSize-(MetaSize+DataSize)$$
## 核心要点
1. 将大 value 限制在单一物理块内，可减少 HDD seek 与尾延迟。
2. 代价是 padding 引入的有界空间放大。
3. 需要了解底层设备和存储系统的真实分配粒度。
## 代表工作
- [[TokaDB]]：把约 1.6 MB 的 KV Cache entry 对齐到 2 MB ByteStore block。
## 相关概念
- [[KV Cache]]
- [[SSD]]
- [[HDD]]
