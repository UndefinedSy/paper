---
type: concept
aliases: [GM, 全局管理器]
---
# Global Manager

## 定义
Global Manager（GM）是 [[K2Plus]] 中负责**跨区域同步控制面**的子系统：管理 schema 与全局数据放置，维护 Global Range View（全局路由表）。

## 核心要点
1. **元数据极简**：只记录按表定义切分的区域键（Global Key range → 主区域 + 副本区域），细粒度键管理下放给各区域的 [[Regional Manager]]。这让 GM **不在事务关键路径上**，不易成为集中式瓶颈。
2. **只被两类事件触达**：(i) 缓存未命中（如冷启动后）；(ii) 低频控制面事件（schema 变更、global range view 更新、区域 failover 编排）。稳态下所有读写请求走 FE/RM 缓存的全局 range view。
3. **缓存正确性靠 epoch**：GM 先持久化 View[$e+1$] 再原子推进 `global_epoch`；RM 给缓存视图打 $e_{local}$ 标签并周期校验（promised TTL 200 ms，租约期内 GM 不更新视图）；epoch 不匹配则刷新，超 TTL 无法校验则保守拒绝请求。
4. **可用性影响面**：GM 不可用只影响控制面敏捷性，不影响数据面可用性——数据面继续用 RM 缓存的 epoch 版本化元数据服务请求；GM 用 [[ZooKeeper]] 做 leader 选举与强一致元数据存储。
5. **实测**：峰值 TPC-C 下 >99.8% 数据面请求走缓存，GM CPU 利用率 < 43%。

## 代表工作
- [[K2Plus]]：GM/RM 两级分区管理是其第一项核心贡献。

## 相关概念
- [[Regional Manager]]
- [[Replication Middleware]]
- [[ZooKeeper]]
- [[Epoch]]
