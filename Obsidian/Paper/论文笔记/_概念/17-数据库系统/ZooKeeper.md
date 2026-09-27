---
type: concept
aliases: [ZooKeeper, Apache ZooKeeper]
---
# ZooKeeper

## 定义
ZooKeeper 是 Apache 的分布式协调服务，提供强一致的元数据存储与 leader 选举等原语。在 [[K2Plus]] 中它承担全局管理器（[[Global Manager|GM]]）的领导者选举与全局视图的强一致存储。

## 核心要点
1. **在 K2+ 中的两个职责**：(i) GM 进程故障时通过 leader 选举自动接管；(ii) 以强一致性序列化 global view change。
2. **commit epoch 机制**：GM 在 ZooKeeper 中先持久化 View[$e+1$]，再原子推进 `global_epoch` 为 $e+1$；[[Regional Manager]] 用 $e_{local}$ 标签校验缓存视图（promised TTL 200 ms，租约期内 GM 不更新全局视图）。epoch 不匹配则刷新，超 TTL 无法校验则保守拒绝请求。
3. **元数据量被刻意压小**：因为物理分区信息不跨区域复制（见 [[Replication Middleware]]），ZooKeeper 只需管理 GM 的全局视图，而非所有区域的细粒度分区元数据——这是 K2+ 让 GM 卸载到非关键路径的前提。
4. **故障影响面可控**：GM 短暂不可用期间，数据面继续用 RM 缓存的 epoch 版本化路由元数据服务请求，仅控制面操作暂停。

## 代表工作
- [[K2Plus]]：用 ZooKeeper 协调 GM 的选举与全局视图版本推进。
- [[Spanner]] / [[Chubby]]: 同类协调服务在工业界的另一条实现路线。

## 相关概念
- [[Global Manager]]
- [[Regional Manager]]
- [[Epoch]]
- [[Leader Election]]
