---
title: "K2+: A Multi-Region OLTP Database with Disaggregated Storage for High Availability and Strong Consistency"
method_name: "K2+"
authors: [Haoze Song, Guoli Wei, Xusheng Chen, Yazhi Feng, Xieyun Fang, Yong Men, Hao Feng, Yongkun Li, Heming Cui]
year: 2026
venue: PVLDB 19(12)
tags: [multi-region-database, disaggregated-storage, replication-middleware, oltp, true-time, high-availability, huawei-cloud]
zotero_collection: VLDB/2026
image_source: local
created: 2026-09-24
---

# 论文笔记：K2+: A Multi-Region OLTP Database with Disaggregated Storage for High Availability and Strong Consistency

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | The University of Hong Kong、USTC、Huawei |
| 日期 | 2026，PVLDB 19(12): 3860–3873 |
| 项目主页 | — |
| 对比基线 | [[TiDB]] v6.4、[[CockroachDB]] v24.1、[[FoundationDB]] v6.0、MySQL (Async)、TiDB + TiCDC、CockroachDB Read Replica、[[MongoDB]] |
| 链接 | [DOI](https://doi.org/10.14778/3827998.3828001) |
| 原始 PDF | `VLDB/2026/K2+ A Multi-Region OLTP Database with Disaggregated Storage.pdf` |
| 前置工作 | [[K2]]（PVLDB 18(6), 2025） |

---

## 一句话总结

> K2+ 用「区域本地存储池 + 独立区域提交日志 + 跨区域复制中间件」把复制与分区管理解耦，使各区域能独立 split/merge/放置物理分区，在 3 区域部署下同步复制达 62 Ktps、异步达 87 Ktps，并已在华为云生产落地。

---

## 核心贡献

1. **全局/区域两级分区管理**：全局元数据保持最小化（仅记录按表定义切分的区域键），细粒度键管理下放给各区域管理器（RM），让每个区域独立 split、merge 与放置分区。
2. **解耦的复制中间件**：在存储池之上引入独立的 Replicator 层，读 DFV 中的 WAL 并跨区域传输；把跨区域复制从请求服务数据节点上剥离，主副本与副本分区形成 N-to-M 对应关系而非强制一对一。
3. **持久化与事务协议协同设计**：提出 [[Persist-in-Progress|PIP 状态]]来串行化竞争性的持久化请求，处理异步网络下的 [[False-Positive Write|假阳性写入]]，并据此区分「必须重试」与「有界重试即可」的持久化点。
4. **生产经验与实测**：在华为云 3 区域部署上给出 TPC-C/YCSB 性能、节点/AZ/区域三级故障恢复数据，以及两个真实业务（云资源用户管理、图数据处理）的落地结果。

---

## 问题背景

### 要解决的问题

云计算的存储计算解耦（storage disaggregation）与地理分布式部署（geo-distribution）两大趋势需要同时成立：既要计算弹性独立于存储池，又要跨区域强一致与容灾。但**把两者真正结合**——在存储解耦架构上做多区域 OLTP——仍很困难，核心难在事务一致性与跨区域复制的协同。

### 现有方法的局限

论文把现有方案归纳为「复制与分区管理紧耦合」（见 Figure 1a）：

| 系统 | 复制方式 | 局限 |
|---|---|---|
| [[Spanner]] | Multi-Paxos 跨区域复制 + Colossus 存储解耦 | 副本按 shard/range 粒度绑定共识组 |
| [[CockroachDB]] | per-range [[Raft]] group | 协调开销随事务涉及的 range 数增长 |
| [[FoundationDB]] | TLog 与存储节点解耦，但 commit proxy 强制全局提交序 | commit proxy 在压力下成为瓶颈 |
| Aurora DSQL | active-active 同步复制 + witness 区域 | 区域内计算存储解耦，跨区域仍是全局 quorum |
| [[TiDB]] | 集中式 PD 负责数据定位与授时 | 跨区域部署时形成区域级瓶颈 |

共同缺陷是：**当副本位于另一区域时，split/merge/migration 等分区操作可能涉及全局协调或全局元数据同步更新**，限制了区域自治性与倾斜负载下的响应能力。

### 本文的动机

K2+ 建立在两块既有基座上：(1) **K2** —— 基于 [[TrueTime]] API 协调、保证跨地理分布节点可串行化的多区域事务协议；(2) **DFV** —— 华为云区域本地存储池，提供弹性、高可靠、高可用的区域内存储，使计算节点无需预留专用盘即可共享存储。

但这两块基座**原生不支持跨区域存储解耦**。K2+ 的思路是在计算节点与存储层之间插入一层复制中间件（Figure 1b），把「本地分区管理」与「全局数据复制」干净分离：区域可以按本地资源与负载模式放置分区，跨区域复制也无需强加物理布局约束，从而在不牺牲正确性的前提下同时获得灵活性、效率与自治性。

---

## 方法详解

### 系统架构

K2+ 采用 **分层解耦架构**（Figure 4），数据路径为红色、控制路径（含数据放置）为蓝色：

- **Driver**：应用接入层，查询语言基于 CQL 协议并支持事务。
- **Frontend**：负责查询解析、事务协调与 QoS 流控；通过 RM 定位数据并路由读写请求。
- **[[TrueTime|TTC Oracle]]**：各区域内跨 AZ 多副本部署，可用性 > 99.9999%；若某区域在超时内无副本能授时，该区域**暂停接纳新的读写事务**以保证正确性。
- **[Data Node]**：以 [[LSM-tree]] 存储数据，key 为表主键、value 为行数据；额外维护 transaction recorder、[[Read Tracker]]（RT）与 [[Write Stream]]（WS），用 B+ 树索引以加速查找。
- **[[DFV|存储池（DFV）]]**：数据节点经 Plog client 持久化，存两类数据——[[WAL]]（事务提交历史）与 LSM 各层压缩数据（用于节点故障后快速恢复）。
- **Replicator**：复制中间件核心组件，从 DFV 读分区 WAL 并推送到其他区域的接收端。
- **Global Manager（GM）**：同步跨区域子系统，负责 schema 管理与全局数据放置；**不在事务关键路径上**。
- **Regional Manager（RM）**：把分配的全局数据进一步切分并映射到本区域数据节点，与所有节点保持心跳；多 AZ 采用 quorum 复制，通过 view change 恢复。

### 模块 1：DFV 存储池与 Plog 抽象

**设计动机**：为存储解耦提供一个「区域内可靠」的落地点——DFV 是华为云统一数据存储平台，支撑对象/块/文件存储到数据库与大数据分析的全部数据类服务。

**具体实现**：
- 存储节点横跨 3 个 [[Availability Zone]]，提供 **[[Plog]]（persistent log）**抽象：一段连续、append-only 的存储区段，通过 Plog client 访问。
- 单 AZ 内靠多副本 + 后台 [[Erasure Coding|纠删码]]保证韧性；跨 AZ 靠**同步写三副本**保证冗余。这是一种刻意的**持久性–可用性权衡**：DFV 在三 AZ 全部落盘后才 ACK，比 quorum 方案提供更强的区域持久性；存储节点故障期间该组的写入会短暂阻塞，但活副本继续提供读。生产指标：**11 个 9 的持久性（99.999999999%）与 5 个 9 的可用性（99.999%）**。
- **可移植性**：K2+ 只要求区域存储池提供两个能力——(1) append-only 持久日志抽象；(2) **[[Seal|sealing 原语]]**（阻止对日志段的后续写入）。Azure Append Blob、[[Delos]] 等系统提供类似 fencing 原语；消融实验（§6.3）表明换成简单的本地文件后端 K2+ 依然功能完整，DFV 主要贡献延迟与运维健壮性。

### 模块 2：事务协议与四个持久化点

K2+ 采用改造后的 [[MVTO|多版本时间戳排序]]协议，借助 [[TrueTime]] 协调事务。Figure 3 展示一次事务的消息流：

- 每个事务在执行前被赋予唯一时间戳；时钟良好同步时，时间戳全局单调递增。
- 用 [[Read Tracker]]（RT，记录曾读过该键的最大事务时间戳）与 [[Write Stream]]（WS，记录所有在途写及其创建者时间戳）检测冲突：**违反读序的写被中止，绕过未提交写的读被阻塞**直至该写提交或中止。
- 成功的读更新 RT；成功的写创建 provisional WS 条目并**立即持久化**（①）以防节点故障丢写。
- 提交阶段记录并持久化事务提交状态（②）；为优化常见路径，写入**异步 finalize**（③）——把 provisional WS 条目转为永久版本是在向客户端报告提交结果之后做的，因为写的最终状态总能由 provisional WS + 事务最终结果重建。
- 写入完全 finalize 后丢弃内存记录，再把事务状态由 committed 持久更新为 finalized（④）。

**关键洞察**：四个持久化点的正确性要求不同——①② 是正确性攸关（提交则必须可恢复），③④ 只影响后续读性能且可从已提交状态重建。这一区分为 §5.1 的持久化策略奠定基础：**正确性攸关点必须重试/恢复不确定的持久化，而 finalize 点有界重试即可**。

### 模块 3：数据分片与物理分区

**设计动机**：K2+ 基于 [[Seastar]]（事件驱动 C++ 协程框架），Seastar 按 CPU 核分资源（每核独立内存）以避免跨核争用，K2+ 沿用该模型把每个核绑定为一个 [[Data Shard|data shard]]。但核间内存空间不同，导致动态负载下 resharding 代价高昂。

**具体实现**（Figure 5）：在 shard 与 [[Physical Partition|物理分区]]之间加入路由层。物理分区（表的部分行集合）是最小管理单元，每个 shard 可容纳多个物理分区；每个物理分区从存储视角对应一棵独立 [[LSM-tree]]。为避免存储碎片与路由复杂度过高，K2+ 定义三类物理分区——**large / medium / small 分别消耗 100% / 50% / 25% CPU**，因此每个 shard 最多 4 个物理分区。

### 模块 4：分区路由（全局 + 区域两层）

以 Figure 6 的 DDL 为例：表有 3 列，$(C_1, C_2)$ 为主键，$C_1$ 作为 **Global Key**（主键子集，不要求唯一）。Global Key 是全局放置的粗粒度局部性提示，让应用层局部性语义引导全局 range 的放置——例如纽约的子公司项目管理部门可用 location 属性（如 302）作 Global Key，把主数据副本放在 $R_3$ 区域以改善局部性。若表中记录无强局部性语义，用户可省略 Global Key 与 global range map，此时表被视为驻留在创建它的区域。

Figure 6 中：$C_1 \in (-\inf, 100)$ 的数据存于 R1、副本在 R2/R3；$[100,200)$ 存于 R2、副本在 R1/R3；$(200,\inf)$ 存于 R3、副本在 R1/R2。

**两级路由**：
1. **全局路由**（Figure 7）：SQL 经 frontend 解析后，写请求定向到持有主副本的区域，读请求选择最近的持有主副本或副本的区域。全局路由由 Global Key 决定、以路由表形式存于 GM，并缓存在 FE 与 RM 上；schema 变更不频繁，故绝大多数请求走缓存不触达 GM。
2. **区域路由**：请求进入区域后按主键剩余部分（本例中的 $C_2$）经 regional range view 转发，最终转发到对应 CPU 核，并携带 **Root Plog ID** 作为持久化层入口；每个 Root Plog 关联若干 Index Plog，记录 WAL Plog 与 LSM 各层的偏移。

**缓存正确性**由 ZooKeeper 中的 commit epoch 保证：GM 先持久化 View[$e+1$] 再原子推进 `global_epoch` 到 $e+1$；RM 给缓存视图打上 $e_{local}$ 标签并周期性校验（配置为 200 ms 的 promised TTL，租约期内 GM 不更新全局视图）；epoch 不匹配则刷新视图后再路由；超出 TTL 仍无法校验则保守拒绝请求。

### 模块 5：复制中间件（本文核心）

**设计动机**：实现存储层与数据节点的**全局解耦**。关键手法是——**物理分区信息仅在其所属区域内可见**，RM 管理的分区元数据不跨区域复制。这既大幅减少 ZooKeeper 需管理的元数据量（只需 GM 的数据视图），又解耦了不同区域的分区策略。

**具体实现**（Figure 8）：
- **N-to-M 对应**：对每个 table range，分区在各区域内独立切分与管理，因此主区域与备区域的分区对应关系是 N-to-M 而非严格一对一。图中 R1 有两个数据节点各管 2 个分区（共 4 个），R2 只有 1 个数据节点管 1 个分区。
- **Replicator Manager**：独立组件，为每个主区域分区维护一份存储在副本区域的 Plog 备份列表（即三条 Rep. Stream）。由于主区域分区可能在复制过程中 split/merge，它**保证任何新分区谱系开始复制前，所有前驱分区都已复制完成**；为此主节点发生 split/merge 时会事先通知 RM 与 replicator manager。
- **复制模式**：同步/异步可按 global key range 或表独立配置。异步模式下主数据节点落盘 DFV 后立即响应；同步模式下主数据节点挂起请求（对应 §2.3 的四个持久化点）直到收到跨区域复制完成确认。**关键设计**：复制并非等数据节点向客户端响应后再发 ACK，而是**在副本区域 WAL 落盘后立即确认**，从而降低跨区域复制延迟；并做**包聚合**——把同一主数据节点内不同分区的确认合并，每条 ACK 携带 Plog ID 与已复制的 WAL 偏移，同一分区的请求只以最大偏移通知。
- **去重**：接收端为每条 WAL stream 跟踪最高成功应用偏移；若因网络重试或 failover 从更早偏移恢复，低于该偏移的记录一律跳过。
- **数据重放**：副本区域数据节点异步重放 WAL 更新 memtable 与 LSM。正确性由两点保证——(1) **可见性由事务协议产生的 [[Epoch]] ID 控制**，WAL 记录创建时打上 epoch 标签，仅当某 epoch 内所有操作重放完毕该 epoch 才对用户请求可见，重放时过滤掉此前已重放过的 epoch 以保持单调可见性；(2) **同 epoch 内重放幂等**，一旦事务或写到达 committed/finalized 就不可回退（Figure 9、Figure 10），因此重复应用同一 WAL 记录只是重复一个已确定的转换。结论：**failover 时可在任意 epoch 内任意点恢复重放**。
- **Replicator 故障切换**：新 replicator 依据 replicator manager 记录的 Plog ID 与已完成偏移续传；数据节点 failover 时对应 replicator 暂停直至同区域新主节点恢复；副本数据节点故障不影响复制进度。

### 模块 6：强一致的保障边界

- **同步复制**下 K2+ 保证所有数据节点上严格[[Serializability|可串行化]]访问，手段是强一致的请求路由 + 基于 ACK 的复制中间件。
- **异步复制**下提供**有界陈旧度的快照读**。
- 正确性验证：论文在 [32] 中对复制协议做了形式化分析，并用 **TLA+ 模型检验**验证关键安全不变式（no dual decision / no lost committed writes）于 [7]。

### 模块 7：驯服异步持久化（PIP 状态）

**问题场景**：与 DFV 的 Plog client 交互经异步网络，可能产生**假阳性失败**——数据看似已持久化但未返回确认。具体异常：事务 $txn_1$ 即将提交，协程 $co_1$ 在 recorder 数据节点发出提交请求并等待持久化成功，但该请求在网络中被阻塞延迟；同时协程 $co_2$ 因 frontend 与数据节点间心跳超时试图中止 $txn_1$，中止请求成功写入 WAL 并触发回滚清理其他节点的临时写；随后延迟的 $co_1$ 请求恢复并把 $txn_1$ 状态作为 **committed** 持久化进 append-only log——于是「虽然 $txn_1$ 已被中止，最新状态却是 committed」。

**解法**：引入新状态 **[[Persist-in-Progress|Persist-in-Progress (PIP)]]**，**不依赖 OS 级锁**地串行化竞争性持久化尝试。PIP 表示系统已发出持久化请求但结果未知，在不确定性消除前必须约束冲突的状态转换。PIP 应用于事务生命周期的全部四个持久化点，形成两个状态机：

#### 7.1 事务状态状态机（Figure 9）

正确性不变式：**若事务最终提交，则提交决定必须已被成功持久化**。

| 状态 | 含义与转移 |
|---|---|
| `InProgress` | 事务活跃。收到提交请求 → `CommitPIP`；否则 → `Aborted` |
| `CommitPIP` | 提交决定已发出、持久化进行中。**必须重试直至收到有效响应（成功或失败）而非未知结果**；`persist_success` → `Committed`，`persist_fail` → `Aborted` |
| `Committed` | 已持久提交。仅当事务写的异步 finalize 完成后 → `FinalizedPIP` |
| `FinalizedPIP` | 事务写 finalize 完成，状态持久化变更进行中；成功后 → `End` |
| `Aborted` | 终态 |

图上还含 `retention_window_expire`、`heartbeat_expire` 触发的 force_abort 转移，以及 `Any_decision (commit/abort)` 标注。

#### 7.2 写操作状态机（Figure 10）

正确性不变式：**若事务最终提交，则临时写必须已持久化**，以防节点故障时数据丢失。

| 状态 | 含义与转移 |
|---|---|
| `InProgressPIP` | 写已创建但尚未持久化。若事务中止 → `InProgressPIPAborted`；持久化成功 → `InProgress` |
| `InProgress` | 写已创建，是否提交未知；按决定转移 |
| `InProgressPIPAborted` | 持久化期间被中止，自动 → `Aborted` |
| `Aborted` | 被中止写的终态 |
| `Committed` | 写已提交 |
| `FinalizedPIP` | 写可 finalize，持久化进行中；**K2+ 对此请求只做有限次重试**，重试后结果仍不明则安全继续，把问题留给后续读写冲突解析阶段处理 |

**整体保证**：每个事务只有一个提交或中止决定生效；延迟或重试的持久化请求不能覆盖已确定（finalized）的决定。

### 模块 8：节点故障切换与防脑裂

**数据节点 failover**：K2+ **总是在同区域内恢复**数据节点，依据两个观察——(1) 区域级故障罕见，DFV 提供高可靠高可用，区域内可快速恢复数据；(2) 主副本放置由客户负载决定以优化数据局部性，同区域恢复有助于保持局部性。实现上把故障节点的所有数据分区迁移到有容量的新节点，并**始终维护若干空闲的热备节点（hot backups）**以加速恢复与负载均衡。迁移流程（Figure 11）：RM 先更新分区放置信息（①）→ 在欠载节点上构建新数据分区 → 恢复旧 WAL 与最新 SSTable（③）→ 新节点 flush 重建的 SSTable 并以新 WAL 开始服务（④）。**K2+ 不恢复在途事务，而是视其为已中止**——因为 read version tracker（功能类似读锁）并未持久化，强行恢复会削弱一致性保证。

**防脑裂（写）**：故障节点可能进入 zombie 状态。K2+ 借助 DFV 的 **I/O fencing API**：新数据节点接管分区前，对底层 Plog 依次发出 **Seal 操作（Root Plog → Index Plog → WAL Plog）**，阻止旧节点继续写。这一点至关重要，因为新旧节点通过存储池**共享同一份 LSM 状态**以加速恢复：sealing 旧 WAL 保证不再有追加生效、旧写者无法修改 LSM 树，于是 **seal 定义了一个稳定的恢复边界**，实现轻量级的「每个 Plog 单写者」。该机制依赖「区域内存储层完全受管且只能通过暴露的 API 访问」这一假设；对不支持原生 Seal 的日志系统，可在不透明存储 API 之上实现通用 fencing 层，但更复杂且低效（需小心跟踪所有权与在途请求）。

**防脑裂（读）**：读可径直从内存提供而不触及持久层，因此 K2+ 对读请求采用 **[[Lease|lease 检查]]**，且**在读取操作之后执行**，从而有效阻止陈旧数据返回用户。**注意：类似检查不能用于写**——数据节点只能在发出持久化请求**之前**而非之后校验 lease，因此即使请求在 lease 过期前发出，仍可能被网络延迟。

**Frontend failover**：同样不恢复在途事务（视为中止），新事务转发到其他 frontend。

**Manager failover**：RM 与 replicator manager 多 AZ 运行，AZ 故障时另一 AZ 的副本成为主。GM 的可用性影响控制面敏捷性而非数据面可用性：GM 用 [[ZooKeeper]] 做 leader 选举与强一致元数据存储；GM 短暂不可用期间数据面继续用 RM 缓存的 epoch 版本化路由元数据服务请求，仅控制面操作（schema 变更、新放置决定、需要提交新全局视图的 failover 编排）暂停。

### 模块 9：区域故障切换与超时机制

**三类区域故障**：
1. 某区域内两个 AZ 的全部 DFV 实例故障 → 该区域 DFV 池变为只读，需规划区域 failover。
2. 某区域内两个 AZ 的 manager 集群（RM + replicator manager）故障 → manager failure，区域内容数据节点/replicator 可能丢失心跳，需区域 failover 恢复管理能力。
3. 区域大规模故障（如 ≥80% 数据节点因断电崩溃）且无法自动恢复 → 强制执行区域 failover 以维持服务连续。

**恢复区域选择**：选持有该分区 global range 覆盖的**最近区域**，与 K2+ 维持工作负载局部性的策略一致。

**恢复流程**：若 failover 前启用了同步复制则流程直接——按区域内数据节点故障相同的分区迁移语义重构数据分区，差别在于需**先更新 GM 的全局 range view**，再更新 RM 的区域分区位置。异步复制模式下部分新写可能未跨区域复制，此时**不可避免地会丢失部分数据，但保证 RPO ≤ 10 s**。核心挑战在于把数据恢复到一个**既保持事务写原子性又保持事务全局序**的快照：K2+ 利用其可见性控制算法的[[Epoch]]分段能力，把受影响分区恢复到某个 epoch，**恢复 epoch 通过协商确定——选择在所有受影响分区上均已完整复制的最新 epoch**。

**超时机制（两类用途）**：
1. **中止无响应事务**：K2+ 支持交互式事务，用户可为每个交互式事务配置最大生命周期；超时后 frontend 向 transaction recorder 发中止请求并关闭与客户端的 session。
2. **清理已中止事务**：节点/frontend failover 时在途事务被视中止且无元数据可恢复，必须靠超时清理。frontend failover 后 transaction recorder 会在超时后触发 `force_abort`（Figure 9 中的 `retention_window_expire`）；数据节点 failover 后其上的在途 transaction recorder 丢失，其他参与者节点会在临时写超时后主动触发 finalize 流程——向新恢复的数据节点查询最终决定，若响应为 `Abort` 或 `Not_Found` 则安全丢弃临时写（`Not_Found` 有两种子情形：在途事务记录已丢失，或事务写已 finalize 且 recorder 已移除，两者都表明可安全删除），若为 `Committed` 则应相应 finalize。

### 模块 10：达成高可用

**可用性定义**（与 [17] 一致）：5 分钟窗口内错误率 > 5%（且请求量 ≥ 60 req/min）即计为 downtime。

**双视角监控**：每个 VM 经 health agent 与心跳向 RM 上报计算用量（CPU、内存、连接数、异常进程状态）；对服务器间请求跟踪延迟与失败率，若超 5% 请求超时或出错持续两个统计周期（如两分钟），RM 告警并在恢复前**隔离受影响组件**。

**过载处理**（Figure 12）：数据节点接近过载（如 ≥90%）时拆分分区并迁移到欠载节点——RM 选择拆分点并更新路由（①）→ 旧分区暂停、把清理过的 segment flush 到 DFV 并阻塞新 WAL 写（②）→ 创建新存储对象（③）→ 既有 SSTable 由新分区的 index Plog 重新索引（④）。随后迁移沿用 Figure 11 流程，但**目的节点跳过 WAL 填充/flush，因为数据已在 ② 阶段 flush**，只复制索引/元数据。实测拆分与迁移在**数秒内**完成（§6.5）。

此外，manager 节点被判不健康时把领导角色重新分配到不同 AZ 的节点；frontend 不健康则加入负载均衡器黑名单，阻止新请求路由过去。

---

## 关键图表

### Figure 1: 现有方案 vs. K2+ 的复制方式对比

![[assets/K2Plus/K2Plus_fig1_motivation.png]]

**说明**：(a) 通过 Consensus Group 复制——按 shard 粒度绑定共识组，副本跨区域时分区操作需全局协调；(b) 通过 Middleware 复制——K2+ 的方案，Shards X/Y/Z 可不同于 Shards 1/2/3，按用户需求与负载均衡独立切分。

### Figure 2: 横跨三可用区的存储池

![[assets/K2Plus/K2Plus_fig2_storage_pool.png]]

**说明**：一个 Geo-Region 内的存储池（持久化层）由 Access Layer（Plog Client + Node Manager）与跨 AZ1/AZ2/AZ3 的存储节点（SSD）构成；DFV 同步写三 AZ 副本后才 ACK。

### Figure 3: 事务协调消息流

![[assets/K2Plus/K2Plus_fig3_txn_flow.png]]

**说明**：Region 1 的 Coordinator / Status recorder / A 的主副本与 Region 2 的 B 主副本之间，经历 Execution（Get TrueTime Timestamp → Read Phase/Write Phase）与 Commit → Finalize 阶段；①persist WS、②persist commit status、③finalize writes、④persist finalized status 四个持久化点标注在图上。

### Figure 4: K2+ 架构总览

![[assets/K2Plus/K2Plus_fig4_architecture.png]]

**说明**：红/蓝两色区分数据路径与控制路径。自上而下为 Driver → Front End Server Cluster（FE×N、CQL Protocol）→ Regional Manager / Global Manager / TTC Oracle / Data Nodes / Replicator → Storage Pool (DFV)；Global Data Placement、Data Location、Progress Monitor、Global Data Partition 属于控制面。

### Figure 5: 数据节点的 Shard 与 Partition 布局

![[assets/K2Plus/K2Plus_fig5_shard_partition.png]]

**说明**：Data Node A 内 Core 0–3 各对应 Shard#0–#3，Partition#0–#7 作为路由层之下的物理分区映射到各 shard；Meta 表记录 `PartitionID | ShardID | TableID | Key_min | Key_max | Size | PLogIDs`。

### Figure 6: 创建 demo 表的 DDL 语句

![[assets/K2Plus/K2Plus_fig6_ddl.png]]

**说明**：定义主键 $(C_1,C_2)$、Global Key $C_1$ 与 global range map：`""::p[R1]::r[R2,R3]`、`"100"::p[R2]::r[R1,R3]`、`"200"::p[R3]::r[R1,R2]`，其中 `p` 表示主区域、`r` 表示副本区域集合。

### Figure 7: 一次读写请求路由的示例

![[assets/K2Plus/K2Plus_fig7_routing.png]]

**说明**：对 `UPDATE demo SET C3 = C3+1 WHERE C1=302, C2=202`，先经 GM 的 Global Range View（`TableID | Key_min | Key_max | Primary_Region | Replica_Region`）定位到 R3，再经 RM(R3) 的 Regional Range View（`TableID | Key_min | Key_max | PartitionID | ShardID | PLogIDs`）定位到 P#1/Shard#1/L1,L2，最终落到 Data Node A 并沿 Root Plog → Index Plog → Write-ahead Plog → Level 0/1/2 Plog 访问 DFV。

### Figure 8: 复制中间件

![[assets/K2Plus/K2Plus_fig8_replication_middleware.png]]

**说明**：Region R1 的 Data Node X（P#0,P#1）与 Data Node Y（P#2,P#3）各自管理 `[-inf,0)/[0,100)/[100,200)/[200,inf)` 的区间分区，经 DFV 的 WAL 由 Replicators 通过 Rep. Stream 复制到 Region R2 的 Data Node Z（P#4，单分区覆盖全域）；展示了主备分区 N-to-M 关系，蓝线为控制路径（ACKs）。

### Figure 9: 事务状态状态机

![[assets/K2Plus/K2Plus_fig9_txn_state_machine.png]]

**说明**：`Initial → InProgress → CommitPIP → Committed → FinalizedPIP → End`（成功路径）与 `Aborted` 终态；含 `Any_decision(commit/abort)`、`persist_success/persist_fail`、`finalize_complete`、`retention_window_expire/heartbeat_expire/abort` 等转移边。

### Figure 10: 事务中写操作的状态机

![[assets/K2Plus/K2Plus_fig10_write_state_machine.png]]

**说明**：`Initial → InProgressPIP → InProgress → Committed / FinalizedPIP → End`；中止分支经 `InProgressPIPAborted → Aborted`；`InProgressPIP` 处 `persist_success/persist_fail` 均可导致 abort。

### Figure 11: 数据分区迁移（故障节点 → 欠载节点）

![[assets/K2Plus/K2Plus_fig11_partition_migration.png]]

**说明**：① RM 更新放置信息；② 新节点接管 P#0；③ 从共享 DFV 恢复旧 WAL 与最新 SSTable；④ 以新 WAL 开始服务。新旧节点共享 LSM 状态是恢复快的原因，也是必须 Seal 旧 Plog 的原因。

### Figure 12: 分区拆分（P#1 → P#1.1 + P#1.2）

![[assets/K2Plus/K2Plus_fig12_partition_split.png]]

**说明**：① RM 选择拆分点并更新路由；② 旧分区暂停、flush 已清理 segment 到 DFV 并阻塞新 WAL 写；③ 创建新存储对象；④ 既有 SSTable 由新分区的 index Plog 重新索引。拆分后的两个分区各持独立的 Root/Index/WAL/LSM。

### Figure 13: 多区域部署 TPC-C 性能（同步跨区域复制）

![[assets/K2Plus/K2Plus_fig13_tpcc_sync.png]]

**说明**：K2+ 与 [[TiDB]] v6.4、[[FoundationDB]] v6.0、[[CockroachDB]] v24.1 的吞吐–平均延迟曲线（横轴吞吐 0–60 Ktps，纵轴对数平均延迟）。K2+ 达 **62 Ktps**；TiDB 在 768 客户端时最早饱和。

### Figure 14: 多区域部署 TPC-C 性能（异步跨区域复制）

![[assets/K2Plus/K2Plus_fig14_tpcc_async.png]]

**说明**：K2+ 与 MySQL (Async)、TiDB (with TiCDC)、CockroachDB (Read Replica) 对比（横轴吞吐 0–80 Ktps）。K2+ 达 **87 Ktps**；MySQL 主从复制把所有写集中到单一 primary 形成瓶颈。

### Figure 15: K2+ 的 TPC-C 可扩展性

![[assets/K2Plus/K2Plus_fig15_scalability.png]]

**说明**：(a) 每区域 4→16 数据节点，吞吐随之扩展，异步优于同步（避免在事务关键路径等待跨区域确认）；(b) 区域数 1→5（每区域 4 数据节点）时吞吐的变化。

### Figure 16: YCSB 敏感性分析

![[assets/K2Plus/K2Plus_fig16_ycsb_sensitivity.png]]

**说明**：(a) Zipf 系数 0.2/0.5/0.8 的影响；(b) 写比例 10%/50%/90% 的影响。更高的冲突率与写比例自然降低性能（更多操作需串行化），但 K2+ 在各客户端数下表现稳健，得益于存储解耦带来的资源弹性与跨节点负载均衡。

### Figure 17: 存储与复制开销的性能影响

![[assets/K2Plus/K2Plus_fig17_overhead.png]]

**说明**：随每节点客户端数增长（1→1024），单 shard 吞吐（KOPS）在 Local File / Using DFV / Async. Replication / Sync. Replication 四种配置下的曲线。这是**论文中最重要的消融图**：DFV 相较本地文件吞吐开销 < 2%，异步复制额外开销 < 3%，同步复制最多降低 42% 吞吐。

### Figure 18: 数据节点故障切换研究

![[assets/K2Plus/K2Plus_fig18_node_failover.png]]

**说明**：5 s 注入节点崩溃 → 心跳超时与分区迁移 → 恢复完成。**主要开销在故障检测**（RM 心跳 + 保守的 5 s 阈值以避免假阳性），10 s 检测到后迁移受影响分区，**恢复到故障前吞吐约需 3 s**。

### Figure 19: 可用区故障切换研究

![[assets/K2Plus/K2Plus_fig19_az_failover.png]]

**说明**：200 s 人为制造 AZ 间网络分区 → 215 s 另一 AZ 的 RM 副本检测到并触发 leader promotion → 260 s 所有数据节点恢复并迁移到新 AZ → 820 s 恢复连通性、原 AZ 的 RM 作为 follower 重新加入共识组 → 1150 s 手动把 leader 角色切回原 AZ（非 failover 必需，用于研究正常 leader 变更期间吞吐）。角色变更时原 leader 吞吐暂降、新 leader 爬升，跨 AZ 聚合吞吐保持稳定。

### Figure 20: 区域故障切换研究

![[assets/K2Plus/K2Plus_fig20_region_failover.png]]

**说明**：GM 确认区域故障后（110 s）副本立即提升为主；对**非活跃分区**则需 [[Cold Start|cold-start recovery]]，额外约 **27 s** 用于重建内存组件（memtable、索引）并向 manager 注册。该额外开销是资源效率权衡（假设区域故障罕见）；SLO 攸关分区可在备区域固定为 always-active 以避免 cold start。

### Figure 21: 扩容与迁移研究

![[assets/K2Plus/K2Plus_fig21_scaling_out.png]]

**说明**：当负载超过 Node B 处理能力时，K2+ 用故障前探测识别不健康节点并施以流控以保住过载节点的 goodput；热点不消退则 RM 触发跨数据节点负载均衡或新增节点。图中标注 Traffic Control 与 Sub-health Alter 两个动作点，并对比 Received Request (Total) 与 Executed Request (Node B)。

### Figure 22: 数据节点间负载均衡研究

![[assets/K2Plus/K2Plus_fig22_load_balance.png]]

**说明**：上图 Node A–D 的 CPU 利用率、下图聚合吞吐，并标注 Splitting and Migration 窗口。Node B 上的分区被拆分并迁移到新 Node A 与欠载的 Node C/D，**聚合吞吐提升 5.7×，拆分与迁移在 3 s 内完成**，均衡后所有数据节点 CPU 利用率超 80%。

### Figure 23: App#1 各事务类型的加速比

![[assets/K2Plus/K2Plus_fig23_app1.png]]

**说明**：云资源管理用户系统对 [[MongoDB]] 的加速比——T1×1.42、T2×6.25、T3×5.29、T4×4.7、T5×10.97；跨区域只读副本加速了访问远端分区的鉴权查询。

### Figure 24: 跨区域可见性延迟（VD）

![[assets/K2Plus/K2Plus_fig24_app1_vd.png]]

**说明**：以 1 s 快照间隔测量，K2+ 达成**平均 521 ms 的可见性延迟**。

### Table 1: App#2 性能因子分析

![[assets/K2Plus/K2Plus_table1_factor_analysis.png]]

| 配置 | FDB | K2+ (DFV w/o LB) | K2+ (DFV) | K2+ (DFV+async. rep) |
|---|---|---|---|---|
| Tput (ktps) | 150.19 | 291.55 | 415.05 | 408.35 |

**说明**：单区域部署下图数据处理服务的吞吐归因。相对 [[FoundationDB]] 的 150.19 Ktps，K2+ 整体 **2.76×** 加速，其中 **1.94× 来自高效事务管理**（K2+ DFV w/o LB 的 291.55/150.19），**1.42× 来自负载均衡**（415.05/291.55），异步复制开销极小（415.05 → 408.35，约 −1.6%）。

---

## 实验

### 实验环境

| 项目 | 配置 |
|---|---|
| 集群 | 24 台机器（scalability 实验除外），默认 3 区域 × 8 机器 |
| 单机 | 8 CPU 核（16 vCPU）、64 GB 内存 |
| 系统 | Ubuntu 20.04、Linux 5.4 内核 |
| 网络 | 实测平均跨区域延迟 **44.2 ms** |
| 资源对齐 | 每区域：1 机器（TrueTime + RM/GM）、1 机器（replicator）、2 机器（frontend）、4 机器（data nodes）；客户端与 frontend 同机 |
| 事务处理核数 | 每区域 64 个（4 数据节点 × 16 vCPU），各系统均 >80% 利用率 |

**基线选择的诚实说明**：论文明确指出工业 OLTP 系统对比本质上无法完美归一——架构、语言、运行时生态（C/C++、Java、Go）与设计范式都不同，精确的「apples-to-apples」对比在实践中往往不可得。为此采取两条原则：确保资源对齐；除跨系统对比外，另提供 K2+ 的系统内分解以隔离存储解耦与复制中间件的影响。

### Workloads

| 数据集/负载 | 规模 | 特点 | 用途 |
|---|---|---|---|
| [[TPC-C]] | 1000 warehouses × 10 districts | 数据均匀分布到 CPU-core shards（partitioned 配置） | 与主流开源 OLTP 系统对比 |
| [[YCSB]] | — | 可调 Zipf 系数与读写比 | 变化负载属性、在 failover 研究中触达峰值吞吐 |
| App#1 用户管理 | 3 DC（乌兰/东莞/深圳）、32 机器、2048 CPU 核 | 鉴权 + 访问控制，mission-critical，要求强一致 | 同步跨区域复制 |
| App#2 图数据处理 | 2 DC、512 CPU 核 | 大规模图构建/遍历/分析，访问模式高度倾斜 | 异步复制 + 动态负载均衡 |

### 主要结果

- **端到端**：异步模式最高 **87 Ktps**，同步跨区域复制达 **62 Ktps**，均高于同设定下的基线。
- **可扩展性**：每区域数据节点 4→16 吞吐随之增长；1→5 区域均能扩展。
- **控制面开销**：峰值 TPC-C 强压 + 全局键重分区压力下，**>99.8% 数据面请求走 FE/RM 缓存的全局 range view**，GM 仅在 cache miss（主要是迁移/重分区过渡期）时被触达，**GM CPU 利用率 < 43%**，非瓶颈。K2+ 正常运行时并不依赖全局键重分区做负载均衡，均衡主要靠存储解耦带来的区域内资源弹性实现。
- **消融（Figure 17）**：DFV 相较本地文件吞吐开销 **< 2%**（实现依赖 [[Seastar]] 的异步调度框架）；延迟上 DFV 追加新数据需 **300 μs**，本地文件 < 100 μs。异步复制吞吐开销 **< 3%**、延迟开销可忽略（复制器只读 WAL 不阻塞数据路径）；同步复制最多降低 **42%** 吞吐、约增加 **45 ms** 延迟（取决于跨区域网络延迟）——论文将其视为容忍区域故障（需要跨区域持久点）的不可避免成本。
- **节点故障（Figure 18）**：主导成本是故障检测（保守 5 s 心跳阈值），检测后迁移到恢复约 **3 s**。
- **AZ 故障（Figure 19）**：15 s 检测 + leader promotion，60 s 内全部数据节点恢复并迁移完成。
- **区域故障（Figure 20）**：副本提升立即完成；**cold-start 恢复额外约 27 s**。
- **弹性（Figure 21/22）**：拆分与迁移 **3 s 内完成**，聚合吞吐提升 **5.7×**，均衡后各节点 CPU >80%。
- **真实业务**：App#1 五个事务类型相对 MongoDB 加速 **1.42×–10.97×**，跨区域可见性延迟平均 **521 ms**（1 s 快照间隔）；App#2 单区域相对 [[FoundationDB]] **2.76×**（1.94× 事务管理 + 1.42× 负载均衡，复制开销极小）。

### 讨论与洞见（Lessons Learned）

- **L#1 让分区自治停留在区域本地**：分层路由策略与复制中间件使每区域独立管理分区——split/merge 与负载均衡成为纯本地操作；跨区域复制只需遵守 WAL 顺序与 split/merge 谱系，分区弹性不再依赖同步的全局元数据更新，显著改善倾斜负载与频繁 resharding 下的运维响应能力。
- **L#2 只让关键分区保持温热**：区域故障罕见，跨区域高可用并非免费——让所有分区在各地都保持 warm 代价过高，而按需激活又会引入 cold-start 恢复时间。K2+ 采用「active-on-demand」策略提升效率，并把关键分区显式 pin 为备区域的 always-active，使权衡有效且保护延迟敏感负载。
- **L#3 高可靠高可用需要全栈协同设计**：生产环境中，多区域 HA 与可靠性只有在端到端工程化与运营时才成立——事务排序、持久性/复制、failover 编排必须共享显式的不变式与故障语义，并辅以故障注入测试与 runbook，让罕见故障变得可预测。

---

## 批判性思考

### 优点

1. **解耦思路清晰且抓住了真痛点**：把「复制」与「分区管理」分离，使物理分区信息区域可见（不跨区域复制元数据），直接消除了 per-range 共识带来的跨区域分区操作同步开销。Figure 22 中「不涉及任何跨区域通信与全局协调即完成重新均衡」是对该设计的直接印证。
2. **PIP 状态机是扎实的工程贡献**：异步网络下 append-only log 上的「延迟提交请求覆盖已中止决定」是一个真实且隐蔽的竞态，PIP 用状态机而非 OS 锁解决，并明确区分「必须重试到明确结果」与「有界重试即可」的持久化点，与四个持久化点一一对应，逻辑自洽。
3. **诚实的评估姿态**：论文主动承认工业系统对比无法完美归一，给出资源对齐原则与系统内消融双轨；且坦承同步复制 42% 的吞吐代价「不可避免」，cold-start 27 s 是「资源效率权衡」，而非粉饰。
4. **生产落地证据**：3 区域 × 32 机器 × 2048 核的 App#1、512 核的 App#2，加上节点/AZ/区域三级故障注入数据，比纯 benchmark 论文更有说服力。

### 局限性

1. **DFV 依赖与可移植性只有定性论证**：Seal 原语是防脑裂的正确性基石，论文承认对不支持原生 Seal 的日志系统需自行实现 fencing 层，但该层「更复杂且低效」的结论并**未给出实现或量化**；消融仅验证了 Local File 后端「功能完整」，未说明如何在 Local File 上实现等价于 Seal 的单写者语义。
2. **异步复制下 RPO ≤ 10 s 的代价被轻描淡写**：区域故障时「不可避免地丢失部分数据」的恢复依赖 epoch 协商回滚到全副本一致的最新 epoch，但论文未给出该协商过程本身耗时，也未讨论多分区并发回滚时的一致性边界。
3. **只读副本的活跃度取舍可能被低估**：cold-start 27 s 对业务而言并非微不足道，而缓解手段「把 SLO 攸关分区 pin 为 always-active」等于把成本转回资源占用，论文未量化在真实负载下有多少比例分区需要 pin、pin 后的资源开销增幅多少。
4. **与基线的对齐仍存在结构性偏差**：K2+ 用 TrueTime 避免了集中式授时（TiDB PD）与 commit proxy（[[FoundationDB]]），但 TrueTime 本身需要每区域 1 台机器 + 跨 AZ 多副本部署；这一基础设施成本在对比中如何折算并不清晰。
5. **缺乏多区域下的能耗/成本维度**：全部评估围绕吞吐、延迟与恢复时间，未涉及跨区域带宽成本——而这在云上多区域部署中往往是主导成本项。

### 潜在改进方向

1. **把分区自治推广到内存层**：论文在 Related Work 中已指出内存解耦（[[LegoBase]]、PilotDB、PolarCXLMem、GaussDB）大多假设单区域 fabric，与地理分布式事务语义的结合仍是开放问题——这恰是 K2+ 复制中间件思路的自然延伸。
2. **自适应同步/异步复制策略**：当前复制模式按表或 global key range 静态配置，可考虑基于跨区域 RTT 波动与业务 SLO 余量动态切换，避免同步模式在广域网抖动时固定付出 42% 吞吐成本。
3. **cold-start 加速**：既然 27 s 主要花在重建 memtable 与索引，可探索以 epoch 为粒度、在备区域**惰性构建内存索引**的增量方案，而非恢复时一次性重建。
4. **Seal 的通用化 fencoring 层**：在抽象存储 API 之上设计并量化一个通用 fencing 层，将直接扩大 K2+ 的可移植场景（如对象存储、跨云存储）。

### 可复现性评估

- [ ] 代码开源 — 论文未提供
- [ ] 预训练模型 — 不适用
- [ ] 训练细节完整 — 实验环境与参数较完整（24 机器、16 vCPU/机、44.2 ms 跨区域 RTT、1000 warehouses）
- [ ] 数据集可获取 — [[TPC-C]]、[[YCSB]] 均为公开标准 benchmark

---

## 关联笔记

### 基于

- [[K2]]: 本文直接建立于 K2 的 TrueTime 多区域事务协议（PVLDB 18(6), 2025），K2+ 复用其 MVTO 协议与可见性控制/epoch 分段算法。
- [[TrueTime]]: 事务时间戳来源；TTC Oracle 每区域跨 AZ 多副本，不可用时暂停接纳新读写事务。
- [[DFV]]: 华为云区域本地存储池，提供 [[Plog]] 与 [[Seal]] 原语，是 K2+ 存储解耦的物理基座。

### 对比

- [[TiDB]]: 依赖集中式 Placement Driver 做数据定位与授时，跨区域部署形成区域级瓶颈，768 客户端即饱和。
- [[CockroachDB]]: per-range [[Raft]] group 复制，协调开销随事务涉及 range 数增长；K2+ 把复制与 range leadership 分离。
- [[FoundationDB]]: TLog 与事务层解耦，但 commit proxy + resolver 仍强制全局提交序，压力下成瓶颈。
- [[Spanner]]: Multi-Paxos + Colossus，是强一致地理分布式的经典范式，但复制仍绑定 shard 粒度共识组。
- [[MongoDB]]: App#1 的 legacy 基线，K2+ 取得 1.42×–10.97× 加速。
- [[Aurora DSQL]]: active-active 同步复制 + witness 区域，是另一条多区域强一致路线。

### 方法相关

- [[Replication Middleware]]: 本文核心贡献，跨区域 WAL-shipping 层。
- [[Persist-in-Progress]]: 驯服异步持久化假阳性失败的关键状态。
- [[Read Tracker]] / [[Write Stream]]: MVTO 冲突检测的两个核心数据结构。
- [[Epoch]]: 可见性控制与幂等重放的粒度单位。
- [[Seal]] / [[Split-Brain]]: 防脑裂的写路径机制（读路径靠 [[Lease]]）。
- [[Physical Partition]] / [[Data Shard]]: 两级分区布局的最小管理单元与资源绑定单位。
- [[Regional Manager]] / [[Global Manager]]: 两级管理面。
- [[Plog]]: DFV 的 append-only 持久日志抽象。

### 硬件/数据相关

- [[Availability Zone]]: DFV 跨 3 AZ 同步复制，是 11 个 9 持久性的来源。
- [[TPC-C]] / [[YCSB]]: 主要评测负载。
- [[Seastar]]: 底层事件驱动 C++ 框架，按核分资源、每核绑一个 data shard。

---

## 速查卡片

> [!summary] K2+: A Multi-Region OLTP Database with Disaggregated Storage
> - **核心**: 用复制中间件把跨区域复制与区域本地分区管理解耦，使各区域独立 split/merge/放置物理分区，物理分区元数据不跨区域复制
> - **方法**: TrueTime + MVTO 事务协议（[[K2]]）+ [[DFV]] 区域内存储解耦 + WAL-shipping 复制中间件 + [[Persist-in-Progress|PIP]] 状态机 + [[Seal]] 防脑裂
> - **结果**: 同步 62 Ktps / 异步 87 Ktps；DFV 开销 <2%、异步复制 <3%、同步复制 −42% 吞吐；节点故障恢复 3 s、AZ 故障 60 s、区域 cold-start +27 s；弹性 rebalance 3 s 内提升 5.7× 吞吐；App#2 相对 [[FoundationDB]] 2.76×
> - **代码**: 论文未提供开源链接

---

*笔记创建时间: 2026-09-24*
