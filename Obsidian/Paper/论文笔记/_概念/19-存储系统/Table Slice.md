---
type: concept
aliases: [表切片, Slice]
---
# Table Slice
## 定义
[[Aurora Limitless Database]] 在 shard 之下引入的细粒度分区抽象：把每个 shard 持有的 shard key 哈希范围进一步细分，典型为每张表 512 个 slice。
## 核心要点
1. shard 内部把自己那部分分片表表示为一张 PostgreSQL 分区表，**分区即对应 slice**——因此分片表整体存在**两层分区**（router 层按 shard，shard 层按 slice）。
2. slice 是 shard split 时的**最小迁移单元**，比整表或整哈希范围灵活得多。
3. **共置表的对应 slice 一起迁移**，从而在扩容后仍保留 [[Data Co-location]] 带来的 join 本地性。
4. standard 表永不分裂，始终留在同一 shard。
5. 分裂点选择：手动分裂时对半切（迁走一半 slice）；自动分裂时依据 heat-monitoring 提供的**热点 slice 信息**决定切分点。
6. 迁移后两侧都会临时持有不属于自己的 slice，靠后台清理（DETACH PARTITION + 删除）收尾。
## 代表工作
- [[AuroraLimitless]]：本抽象的提出与实现。
## 相关概念
- [[Sharding]]
- [[Data Co-location]]
- [[Copy-on-Write Cloning]]
- [[Partition Pruning]]
