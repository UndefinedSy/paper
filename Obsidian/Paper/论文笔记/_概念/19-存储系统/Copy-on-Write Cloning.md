---
type: concept
aliases: [写时复制克隆, CoW Clone, 卷克隆]
---
# Copy-on-Write Cloning
## 定义
存储层提供的卷克隆能力：创建克隆时不实际复制数据，仅共享底层页面，只有当任一侧发生修改时才对相应页面做复制。
## 核心要点
1. 克隆是 O(1) 的元数据操作，与卷大小无关，因此创建极快且初始不占额外空间。
2. [[Aurora Limitless Database]] 用它作为 shard split 的第一阶段：克隆源 shard 的卷而非从源 shard 读数据再写新 shard。
3. **核心价值：把数据搬迁开销卸载到存储层**，从而「即使 shard 已在满载运行也能被分裂」——直击自动分片系统最尴尬的死锁场景（越需要分裂的节点越没余力做分裂）。
4. 克隆需要时间，期间源 shard 仍在写入，故须接一个 [[WAL|redo 日志]]回放阶段来追平。
5. router 的新增也复用同一机制（克隆一个既有 router）。
## 代表工作
- [[AuroraLimitless]]：shard split 与 router 新增的基础。
- [[Amazon Aurora]]：卷克隆能力的提供者。
## 相关概念
- [[Aurora Distributed Storage]]
- [[Table Slice]]
- [[Amazon Aurora]]
- [[WAL]]
