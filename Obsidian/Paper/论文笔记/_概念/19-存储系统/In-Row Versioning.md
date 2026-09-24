---
type: concept
aliases: [in-row 版本, In-row Versioning]
---

# In-Row Versioning

## 定义

[[Persistent Version Store]] 的一种优化形态：把行的旧版本作为 diff 直接嵌入当前版本所在的同一数据页内，而不是存到独立的版本存储区。

## 核心要点

1. 由于更新通常只改动少数列，只需存储新旧版本之间的差异（diff），省去访问额外页来重建旧版本的开销，同时降低写路径的页 IO 与日志开销。
2. 生成 in-row 版本会增加行大小，可能诱发 B-Tree 页分裂（代价较高、会退化索引结构），因此系统对 diff 大小与行大小都设了上限。
3. 与 [[Off-Row Versioning]] 互补：超出 in-row 限制的版本转入 off-row 存储，SQL Azure 同时维护两者以平衡读性能与写放大。

## 代表工作

- [[AzureMVCCVersionCleaner]]：论文以 off-row 版本为主要讨论对象，但强调所有优化同样适用于 in-row 版本。
- Constant-Time Recovery（VLDB 2019）：in-row 版本存储的原始设计来源。

## 相关概念

- [[Off-Row Versioning]]
- [[Persistent Version Store]]
- [[Constant Time Recovery]]
