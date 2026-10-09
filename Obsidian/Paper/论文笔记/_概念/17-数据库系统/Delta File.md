---
type: concept
aliases: [delta file, delta 文件]
---

# Delta File

## 定义

[[Delta-Base Architecture|delta-base 架构]]中承接实时写流量、临时存放增量数据的轻量文件，在写入一定量或经过一定时间后被 `flush` 合并进基索引。

## 核心要点

1. **组织方式**：内部是 [[Ring Queue|环形队列]]，插入轻量、写延迟低，并可吸收突发写流量。
2. **分区存储**：向量与其对应的上下文标识 `ctid` **连续但分开**存储，便于利用 [[SIMD]] 等指令级并行做硬件加速。
3. **双路径召回**：查询同时探测 delta 文件与基索引，数据写入 delta 后**立即可见**，保证实时新鲜度。
4. **冲突收敛**：把写—写冲突（并发 insert 之间、insert 与 `vacuum` 之间）限制在文件内部，从而使基索引扫描可以**无锁**执行。
5. **并发与多实例**：delta 文件自身用读写锁；Nova 默认维护**三个** delta 文件提升并发度。从 delta 读取有额外开销，但相对基索引无锁收益可忽略。
6. **容量影响**：实验显示 delta 大小从 1000 增至 10000 向量时 QPS 从约 6400 降至约 3700，因此需要自动 flush（默认阈值 5000 向量，由写热度与大小共同驱动）。
7. **保留原始向量**：启用量化时基索引只存量化向量，delta 文件保留原始向量以便精确重算。

## 代表工作

- [[Nova]]: [[Nova Memory|NovaM]] 与 [[Nova Disk|NovaD]] 均以此为写入层；对比 [[ADB-V]]（无 delta 层，直接插入基索引）的写延迟差距量化了该设计的价值。

## 相关概念

- [[Delta-Base Architecture]]
- [[Ring Queue]]
- [[WAL]]
- [[Lazy Index Construction]]
