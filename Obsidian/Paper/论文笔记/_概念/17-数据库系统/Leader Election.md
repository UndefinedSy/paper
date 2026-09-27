---
type: concept
aliases: [Leader Election, 领导者选举]
---
# Leader Election

## 定义
Leader Election 是在分布式系统中从一组候选副本中选出**唯一主副本（leader）**来承担写职责的过程，通常由共识协议（Raft / Paxos / ZooKeeper ZAB）驱动。

## 核心要点
1. **触发时机**：leader 进程崩溃、网络分区、或原 leader 被判定不健康（如 [[Split-Brain]] 风险）。
2. **[[K2Plus]] 中的应用场景**：
   - [[Global Manager|GM]]：用 [[ZooKeeper]] 做 leader 选举；切换期间数据面继续用缓存路由元数据，仅控制面暂停；
   - [[Regional Manager|RM]] 与 replicator manager：跨 AZ quorum 复制，AZ 故障时另一 AZ 的副本经 leader promotion 接管（实测 15 s 内检测到并完成提升）；
   - **正常 leader 变更**：论文在 AZ 故障实验中手动把 leader 切回原 AZ，观察到原 leader 吞吐暂降、新 leader 爬升，而**跨 AZ 聚合吞吐保持稳定**。
3. **与 fencing 的配合**：选举出 leader 还不够，必须防止旧 leader 继续写（zombie）。[[K2Plus]] 靠 [[Seal]] 封禁旧日志段实现单写者。
4. **选举延迟直接影响可用性**：故障检测阈值（如 K2+ 保守的 5 s 心跳阈值）在「假阳性风险」与「切换速度」之间权衡。

## 代表工作
- [[K2Plus]]：GM 依赖 ZooKeeper 选举；RM/replicator manager 在 AZ 故障时由另一 AZ 副本晋升。
- [[Spanner]] / [[CockroachDB]]：以 Multi-Paxos / Raft 实现 range 级 leader 选举。

## 相关概念
- [[ZooKeeper]]
- [[Regional Manager]]
- [[Global Manager]]
- [[Seal]]
- [[Availability Zone]]
