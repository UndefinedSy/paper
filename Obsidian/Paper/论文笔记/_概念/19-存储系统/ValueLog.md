---
type: concept
aliases: [Value Log, 值日志, VLog]
---
# ValueLog
## 定义
在 KV separation 中独立、通常追加写地保存大 value 的日志文件；LSM-tree 或其他索引只保存 key 与 value 地址。
## 核心要点
1. 减少大 value 参与 compaction 导致的写放大。
2. 读取需先查索引再访问 ValueLog，可能形成 double-read。
3. 更新和删除产生无效空间，需要垃圾回收。
4. 相关 value 的空间局部性直接影响点读 I/O 次数。
## 代表工作
- [[TokaDB]]：按用户与时间 segment 组织 ValueLog，并用 LTGC 恢复局部性。
## 相关概念
- [[KV Separation]]
- [[LSM-tree]]
- [[Locality-Triggered Garbage Collection]]
