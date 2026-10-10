---
title: "Salesforce Database: A Cloud-Native Multi-Tenant OLTP Database"
method_name: "SalesforceDB"
authors: [Atish Agrawal, Vaibhav Arora, Gary Baker, Milena Bergant, Benjamin Busjaeger, Subho Chatterjee, Terry Chong, Roy Chowdhuri, Doug Doole, Haiyan Du, Thomas Fanghaenel, Pat Helland, JV Jujjuri, Jim Mace, Ashish Mahajan, Jamie Martin, Nicolas Michael, Ayman Mobarak, Steven Raspudic, Randy Spalten, Simon Wong, Nat Wyatt]
year: 2026
venue: PVLDB 19(12)
tags: [cloud-native-database, multi-tenancy, oltp, lsm-tree, shared-storage, scale-out, salesforce]
zotero_collection: VLDB/2026
image_source: local
created: 2026-10-10
---

# 论文笔记：Salesforce Database: A Cloud-Native Multi-Tenant OLTP Database

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | Salesforce |
| 日期 | 2026，PVLDB 19(12): 4426–4439 |
| 项目主页 | — |
| 对比基线 | [[Aurora DSQL]]、[[Spanner]]、[[PolarDB]]、[[CockroachDB]]、[[Snowflake]] |
| 链接 | [DOI](https://doi.org/10.14778/3827998.3828043) |
| 原始 PDF | `VLDB/2026/SalesforceDB- A Cloud-native Multi-tenant OLTP Database.pdf` |

---

## 一句话总结

> SalesforceDB 把租户作为数据库一等公民嵌入单一租户聚簇 [[LSM-tree]] 存储，叠加计算存储分离与不可变存储，通过"LSM 虚拟化"将租户迁移、克隆、加密等重操作变成元数据级别操作，并以单 Coordinator + 多 Executor 架构实现低协调开销的横向扩展。

---

## 核心贡献

1. **LSM 原生的共享表多租户（Shared-Table Multi-Tenancy）**：用"银河系（galactic）键空间"将每个租户的数据在单一 LSM 内聚簇为连续键区间，既支持单实例承载数百万小租户，又能支撑超大租户独占实例并按需扩展。
2. **LSM 虚拟化（LSM Virtualization）**：引入 [[Virtual File References|虚拟文件引用]]（Extent Reference / ERS），把租户迁移、沙箱克隆、租户级加密等传统需要物理拷贝数据的重操作转化为元数据级常数时间操作。
3. **低协调横向扩展架构**：单 Coordinator 负责写协调和最近提交数据服务，多个 Executor 对等处理读写请求并直接 tail Coordinator 日志，借助 Memstore Bloom Filter 消除约 99% 的探测 RPC，实现近线性扩展而无需分区或两阶段提交。
4. **基于 [[SDBStore]] 的不可变分布式存储**：Fork 自 [[Apache BookKeeper]]，以云对象存储为最终系统记录、分布式缓存提供低延迟访问，quorum 写入跨三个可用区保证 AZ+1 可用性与零数据丢失。

---

## 问题背景

### 要解决的问题

Salesforce 的 CRM 应用是一个高度多租户（当时已有超过 400 万租户）、读多写少（约 30%写/70%读交织在同一事务中）、工作负载高度倾斜且不可分区的事务型系统。数据库层需要同时满足：
- 极高可用性（容忍整个 [[Availability Zone|可用区]]故障、支持无停机升级和补丁）与零数据丢失的强耐久性；
- 原生支持多租户，高效承载数百万小租户及少数需要独占整个实例的超大租户；
- 支持租户级能力：即时创建沙箱副本（用于开发/测试/培训）、租户迁移、按租户独立加密密钥（合规/监管要求）；
- 在商用云基础设施（无专用硬件如 SAN、Infiniband）上运行，并随租户/数据规模线性扩展。

### 现有方法的局限

- 传统数据库若要创建沙箱，需要物理拷贝整个租户数据，大租户耗时极长，拖慢开发周期。
- [[Shared Memory|共享进程]]式多租户（如 [[Relational Cloud|RelationalCloud]]、[[Elastras]]）多租户共享数据库服务器但 schema 隔离度有限；[[CockroachDB Serverless]] 虽解耦节点但租户间不共享 schema，难以做到 SalesforceDB 这种跨数据库统一 schema 迁移。
- 传统基于分区的扩展方案（[[Spanner]]、[[CockroachDB]]）对跨分区事务需要 [[Two-Phase Commit|两阶段提交]]，不适合高倾斜、不可分区的工作负载。
- 传统可变页式存储引擎在页面变更、缓存驱逐时需要重新加密整页，无法高效支持按租户粒度轮换密钥；也更易出现并发修改导致的物理/逻辑损坏。

### 本文的动机

Salesforce 认为，把"租户"作为数据库内部的一等对象、并将租户数据物理聚簇存放，可以让迁移/克隆/加密这些原本是行级或物理级的重操作下沉为元数据操作；同时结合[[Immutability|不可变]]存储和计算存储分离，能用普通云基础设施达到高可用、高扩展性，并避免传统扩展方案依赖的分区或专用硬件。

---

## 方法详解

### 整体架构

SalesforceDB 基于 [[PostgreSQL]] 构建（继承查询处理、事务处理、schema 管理能力），但自研了专有的 [[LSM-tree|LSM]] 存储与访问层以满足扩展性、简化云运维和多租户的目标。核心架构原则：

- **计算存储分离**：无状态 SQL 计算节点从共享分布式存储读取数据，计算和存储可独立扩展；多个数据库实例可共享同一存储集群。
- **不可变性**：采用不可变 [[LSM-tree]] 存储，消除可变引擎中 B-tree 分裂、页修改引发的腐化和并发问题，简化备份恢复，并大幅减少跨节点协调需求。
- **多租户原生支持**：租户是数据库对象，存储按租户聚簇在单一 LSM 内。
- **云原生设计**：运行在商用云基础设施上，[[Kubernetes]] 编排部署与自动化故障恢复。

整体部署如 Figure 1：跨三个可用区部署，每区有应用服务器、一套 DB 集群（一主两备）与缓存集群，底层共享云存储；协调层（图中省略）运行 [[ZooKeeper]]，提供领导者选举、quorum 服务与客户端发现。

### SQL 计算层与一致性保证

SQL 计算节点无状态，所有状态都在存储层；节点随时可被终止，由 [[Kubernetes]] 自动拉起新实例并从存储恢复。schema 变更支持双版本并存（旧/新 schema 同时激活，通过 trigger 对齐更新），DDL 对现有 schema 的修改是非阻塞的，支持并发在线建索引。

写入从不覆盖存储；每次写追加新 block，每个 block 带包含文件 ID 和 block 号的唯一校验和，用于验证读取正确性与防止物理损坏。针对 [[MVCC|快照式 MVCC]] 可能引发的逻辑损坏（基于过期版本的更新），SalesforceDB 在记录中嵌入**lineage（血统）**信息——即前一版本的提交时间戳，更新时校验新版本的血统是否匹配该行最近一次提交的版本，从而拒绝基于过期版本的修改。

### [[SDBStore]]：不可变分布式存储

SDBStore 是 Fork 自 [[Apache BookKeeper]] 的仅追加、非 POSIX 存储系统，以云对象存储（如 [[AWS S3]]）为系统记录，叠加分布式缓存提供低延迟访问。单个 SDBStore 集群可被多个 SalesforceDB 实例共享（Figure 2）。

- 存储模型以 **Extent**（类似文件）和 **Fragment**（类似文件内的 block）组织。Extent 由随机生成的 128-bit UUID 标识，支持跨数据库无协调共享命名空间；Extent 只能有单一写者，经历 创建→追加 fragment→关闭→（不再被引用后）删除的生命周期；fragment 一旦写入不可修改，extent 关闭后不可再追加。
- **双缓存设计**：数据缓存（write-through，基于本地 ephemeral NVMe，针对高吞吐顺序写与低延迟随机读优化）与事务日志缓存（write-back，基于 [[AWS EBS]] 等持久块存储，针对低延迟持久写与高吞吐流式读优化）。两者共享 API 但实现不同，可独立伸缩。
- **日志缓存的 Quorum 机制**：ensemble 大小 $E_n=6$（跨 3 个 AZ），写 quorum $Q_w=6$，确认 quorum $Q_a=4$，读 quorum $Q_r=3$。读 quorum 为 3 保证即使在 AZ+1 故障场景下也能准确判定日志结尾。故障时 ensemble 自动用存活 Bookie 替换失败节点，对数据库透明。
- 云存储随机读的尾延迟比内存缓存慢约四个数量级，因此整个数据库工作集尽量驻留缓存，每个 extent 在缓存中有 3 副本分布于不同 AZ，并使用 [[Hedged Reads|hedged reads]] 进一步降低尾延迟。

### 共享表多租户（Section 4）

SalesforceDB 采用 **shared-table** 多租户模型（而非按租户建库或共享进程模型），以最小化单租户开销，并与 Salesforce 应用层 schema（多租户数据共处同一张表）对齐。

- **租户作为数据库对象**：每个租户有唯一 Tenant ID，拥有生命周期状态——Creating（初始）、Active（可读写）、Quiesced（只读）、Maintain（内部运维）。任何修改租户数据的事务会在租户对象上加共享锁，用于与改变租户状态的操作同步。
- **租户聚簇键编码**：整个数据库使用单一有序 LSM 键值存储，支持线性键空间（称为 **galactic keyspace**，因为含唯一 Tenant ID 使键在整个 SalesforceDB 机群范围内唯一）。多租户表的主键必须以 Tenant ID 作为首列；其辅助索引的索引键也必须以 Tenant ID 为首列。多租户表记录的 galactic key = Tenant ID + Table ID + 剩余主键列；非租户表记录的 key = Table ID + 主键列。索引记录的 galactic key 由 Tenant ID（若适用）、Index ID、索引键和对应基表记录的 galactic key 组合而成。
- 由此带来的性质：租户数据与非租户数据映射到键空间的两个不同区段；多租户表及其索引在线性键空间上是**不连续**的；但**单个租户**在某张多租户表或索引中的数据**连续**。全表扫描（无 Tenant ID 限定）在 Salesforce 被视为反模式，系统会透明地把这种扫描转化为与租户表的 Join。
- **租户隔离**：由应用层添加谓词实现逻辑隔离，数据库在会话设置租户上下文时检查并强制这些谓词存在；性能隔离通过数据库引擎上报资源使用（CPU、Memstore、连接、I/O）给应用层，应用层据此对 Org/User/URI 粒度限流。

Figure 3 对比了传统数据库（按表存储、租户数据跨表混合）与 SalesforceDB（租户数据在表内聚簇连续）的物理组织差异。

### [[Tenant Encryption|租户加密]]（Section 4.3）

租户聚簇 LSM 使得按租户轮换密钥的加密成为可能：每个 fragment 的数据仅属于单一租户，flush/compaction 时按租户密钥加密整个 fragment，密钥 ID 嵌入 fragment 元数据；读取时按嵌入的 key ID 解密。由于加解密是在 fragment（不可变存储单元）边界完成，不会像传统可变页引擎那样因页面驱逐而频繁重新加密。密钥轮换由 [[KMS|密钥管理系统]]触发，经由 compaction 完成数据重加密；轮换完成前，旧密钥加密的数据仍可读取。

### LSM 虚拟化（Section 5）

这是本文的核心创新。SalesforceDB 利用租户聚簇和不可变 LSM，把租户克隆、迁移变成"元数据操作"，称为 **LSM Virtualization**。

#### Virtual File References（虚拟文件引用）

虽然 extent 本身不可变，但 LSM 的"形状"会随 flush/compaction 变化。该形状由每数据库实例的**存储清单（storage manifest）**维护，元数据由两部分组成：
- **Extent Reference（ER）**：描述一个 extent，包含 extent ID 及定义可见键空间（minKey、maxKey）、时间戳范围、租户映射（源租户、新租户）的元数据，是对底层 extent 的抽象层——相当于"虚拟文件引用"。
- **Extent Reference Set（ERS）**：关联一组 ER。

多个 ER 可指向同一 extent，ER 中的元数据可限制可见性到特定键、时间或租户；ER 中的租户映射信息被嵌入访问层，在查询处理和 compaction 时透明执行租户翻译。因为记录按租户聚簇，ERS 可被精确定义以匹配 LSM 中某个租户的全部记录，于是克隆/迁移/删除租户都能通过对存储清单做元数据操作完成。

#### [[Tenant Snapshot|租户快照]]

一个租户的 ERS 可保存进存储清单成为**Tenant Snapshot**；由于 extent 不可变，快照天然是某时刻租户数据的稳定表示，此外还包含数据库状态元数据（如 schema fingerprint）。

Figure 4 展示了租户聚簇如何支持跨多个 LSM 层级组合 Extent Reference 来构建快照：数据库的 ERS 由 ER1~ER10 构成；要为 Tenant B 构造快照，与 Tenant B 键空间相交的 8 个引用被复制为虚拟引用 ER1′、ER2′、ER3′、ER4′、ER7′、ER8′、ER9′、ER10′；与其他租户数据重叠的引用被裁剪限定到 Tenant B 键空间（如 ER1′/ER2′/ER7′的 minKey 被调整到 Tenant B 键空间起点，ER4′/ER10′的 maxKey 调整到其终点）。这 8 个虚拟 ER 共同构成该租户快照。创建、导出、导入、删除租户快照均为**元数据操作**，底层数据 extent 只需保持稳定，无需移动或拷贝——由于元数据比数据小几个数量级，这些操作实质上是常数时间。

#### 跨数据库存储共享的前提条件（Section 5.3）

要跨数据库迁移/克隆租户需保证：
1. **Schema 一致性**：目标库若有源库没有的索引，迁移后查询可能用错索引产生错误结果。SalesforceDB 为每个实例维护**schema fingerprint**（一个捕获表/索引/属性逻辑形状的紧凑哈希值），导入前比对租户快照中的 fingerprint 与目标库 fingerprint 来验证兼容性；
2. **Extent 删除时序安全**：一个 extent 只有在不再被同一存储集群内任何数据库的任何 ERS 引用时才会被删除，应用层负责协调创建/导出/快照的顺序，确保被共享的 extent 在整个共享过程中始终保持被引用状态。

#### 租户迁移（Tenant-Migration, Section 5.4）

由应用层选择共享同一存储集群的兼容源/目标库，两个数据库之间**不直接通信**，全部由应用编排：
1. 应用请求将租户置为 Quiesced（排空进行中写事务，超时未完成则 abort）；
2. 创建租户快照（捕获租户的时间点状态，内存中数据先 flush 到持久 LSM）；
3. 快照的 Extent References 导出到目标库实例；
4. 目标库先创建一个处于 "Creating" 状态（对应用不可见）的空租户，将快照 ER 集合接入目标实例存储清单（将租户"拼接"进目标键空间）；
5. 目标租户激活，应用开始路由客户流量；迁移完成后，源数据库上的源租户和快照被删除；出错或超时则重新激活源租户。

#### 沙箱创建（Sandbox Creation, Section 5.5）

与迁移类似但目标是**克隆**而非移动，需要应用层额外的后处理（如按行删除沙箱不需要的数据、PII 脱敏）：
1. 在源上创建租户快照，传输到承载沙箱的目标实例；
2. 目标实例为沙箱创建新 Tenant ID，将快照 ERS 拼接入新租户键空间；
3. 与迁移不同，这里对所有导入的 ER 添加**源租户→克隆租户**的映射，更新这些引用的 minKey/maxKey 指向克隆租户，而物理 extent 数据仍引用源租户；
4. 查询克隆租户时在运行时基于租户映射做翻译——probe/scan 读到克隆租户键空间内的 ER，底层数据记录携带的仍是源 Tenant ID，访问层将其翻译回克隆租户 ID 返回给调用方（除主 Tenant ID 列外，schema 还支持显式指定其他需要翻译的 Tenant ID 列）；
5. LSM compaction 在有租户映射的键空间上同样执行翻译，写出时直接写克隆租户 ID；沙箱上的任何新写入都直接以克隆租户 ID 编码。

### 计算横向扩展（Section 6）

SalesforceDB 的计算扩展必须同时满足：**外部一致性**（多节点集群行为等价于单一集中式实例）、**读写交织的 OLTP 负载**（不能靠把读发给只读副本来扩展）、**不可分区工作负载**（高倾斜 schema，不能假设数据局部性或应用层分区）、以及**商用云基础设施**（无 SAN/Infiniband 等专用硬件）。

#### 集群组件（Figure 5）

主读写集群由单个 **[[Coordinator]]** 节点和多个 **[[Executor]]** 节点组成：
- **Coordinator**：唯一专职节点，处理所有底层记录写入，服务最近提交数据的读取。托管集群唯一活跃的 Memstore（保存未提交数据、维护行锁、裁决锁冲突），每条未提交记录相当于隐式行锁。写入经 Coordinator 本地事务日志持久化到 SDBStore。Coordinator 同时托管 **Epoch Coordinator**（时间戳 oracle），以 [[Transaction Commit Number|事务提交号（XCN）]]（64 位无符号整数、近似纳秒级系统时间的逻辑时钟）维护两个 epoch：当前提交 epoch（用于分配提交时间戳）与当前可见 epoch（用于以 [[Snapshot Isolation|快照隔离]]运行的语句分配读时间戳）。可见 epoch 保证单调递增且严格小于提交 epoch，大约每 1ms 推进一次并广播给所有集群成员。Coordinator 故障时由 [[ZooKeeper]] 从剩余 Executor 中选举新 Coordinator，典型全故障转移恢复时间 < 5 秒，对客户端透明（RPC 可能短暂停滞但不报错，扫描等操作跨 Coordinator 实例透明恢复）。
- **Executor**：对等节点，接受读写连接，tail Coordinator 的事务日志，本地完成大部分查询规划与执行；只有读取 Coordinator 活跃 Memstore 或写入时才需要 RPC。

#### 写与提交流程（Section 6.2）

DML 由 Executor 本地处理，通过访问方法把记录透明地插入 Coordinator 的 Memstore。由于单行 INSERT 大部分耗时在查询执行本身（生成新记录、校验约束、触发器、生成返回值等），对应 RPC 开销相对整条语句执行只占约 **5%**。SalesforceDB 通过 RPC 批处理（如单行 INSERT 涉及多个辅助索引插入会合并为单个 RPC）、异步发送（多行修改异步提交、语句末尾同步点阻塞）将 RPC 延迟影响降到最低。

提交时 commit RPC 发往 Coordinator，阻塞直到事务在 SDBStore 的事务日志中完全持久化，返回提交时间戳（XCN）。为保证外部一致性，在向客户端返回提交确认前，Executor 需检查本地读时间戳已越过该提交 XCN 并再推进一次，确保集群内后续任何查询都能看到已提交数据——这一额外"提交停顿"仅由提交方承担，且通常很小（持久化耗时 1–3ms 约等于全局 epoch 推进两次所需时间 2ms）。

#### 读流程（Section 6.3，Figure 6）

Salesforce 查询始终以 **[[Read Committed Snapshot Isolation|Read Committed Snapshot Isolation (RCSI)]]** 运行（基于 [[MVCC]]），除 SELECT FOR UPDATE 外几乎不需要 RPC 到 Coordinator，除非需要访问语句快照时间戳要求的极近期提交数据。

- 读分为 **probe**（按唯一键取单条记录）和 **scan**（范围扫描）。probe 按时间顺序逐层查找 LSM（先 Memstore，再逐级 LSM extent），直到找到 commitXCN ≤ snapshotXCN 的记录；scan 则在 Memstore 和各 LSM 层打开子扫描并归并去重。
- 由于约 **99.9%** 的查询数据可通过共享不可变持久化存储获得，节点间无需为持久化访问协调——这与传统页式数据库需要分布式页缓存+跨节点失效+分布式闩管理截然不同；每个 Executor 拥有自己的缓冲管理器，可独立加载读取数据，查询优化器因此不需要在代价估算中考虑远程访问。
- 每个 Executor 通过 SDBStore tail Coordinator 日志，维护从持久化最大 XCN 到自身 **redoXCN**（本地已完全重放进 Memstore 的最近提交时间戳）之间数据的完整视图（Figure 6）；redoXCN 与持久化最大 XCN 之间的 scan/probe 可直接从本地 Memstore 服务。为保证未提交事务总能本地读到自己的未提交写，Executor 也会把事务写入应用到本地 Memstore（并在 redo 过程中与 Coordinator 对账）。
- **Memstore Bloom Filter**：Executor 利用从 Coordinator 获取的 Bloom Filter 判断 Coordinator Memstore 中是否可能存在目标数据，避免不必要 RPC。Coordinator 为 Memstore 中已提交的每条记录（主键及唯一辅助键）维护 Bloom Filter 并推送给 Executor；对于 scan，Coordinator 额外存储被修改关系（索引/基表）的键前缀及辅助索引被修改记录到首列的键前缀。实测 Memstore Bloom Filter 消除约 **99%** 的 probe RPC 和约 **60–70%** 的 scan RPC。

---

## 关键公式

本文偏系统工程论文，核心机制均用文字 + 架构描述而非数学公式表达，论文中未出现形式化的 LaTeX 公式。关键量化关系以文字/参数形式给出，摘录如下：

### SDBStore 日志缓存 Quorum 参数

$$
E_n = 6,\quad Q_w = 6,\quad Q_a = 4,\quad Q_r = 3
$$

**含义**：日志缓存 ensemble（存储节点集合）的规模与 quorum 配置，决定日志写入的持久性、可用性与故障转移语义。

**符号说明**：
- $E_n$：ensemble 大小，即共同存储每个 fragment 副本的 Bookie（存储节点）总数，跨 3 个 AZ 分布；
- $Q_w$：写 quorum，一次写操作需要确认的节点数；
- $Q_a$：确认 quorum，判定一次写已被持久确认所需的最小节点数；
- $Q_r$：读 quorum，日志尾读（判定日志结尾）所需读取并比对的副本数，设为 3 以保证 AZ+1 故障场景下仍可准确判定。

### 外部一致性提交停顿条件（文字公式化表达）

$$
\text{commit\_visible} \iff \text{local\_read\_ts} \;\ge\; \text{commit\_XCN} \;\text{（并再推进一次）}
$$

**含义**：Executor 在向客户端返回提交确认前，必须确认本地可见 epoch（读时间戳）已经超过该事务的提交 XCN 且再推进一次，从而保证集群中任何后续查询都能看到该提交，以此维持外部一致性。

**符号说明**：
- `local_read_ts`（即 visible epoch）：该 Executor 当前可见的读快照时间戳；
- `commit_XCN`：事务提交时被分配的事务提交号。

---

## 关键图表

### Figure 1: SalesforceDB Architecture Overview

![[SalesforceDB_fig1_architecture_overview.png]]

**说明**：三个可用区各部署应用服务器、一套 DB 集群（Primary 在一个区，Secondary 在另两个区）与缓存集群，统一经网络负载均衡器对外，底层共享 Cloud Storage。体现了 [[Separation of Compute and Storage|计算存储分离]]与 AZ+1 冗余设计。

### Figure 2: SDBStore Overview

![[SalesforceDB_fig2_sdbstore_overview.png]]

**说明**：SDBStore 客户端通过日志读写访问 Log Cache（跨 3 AZ，基于 EBS 类块存储，异步写回对象存储），通过数据读写访问 Data Cache（跨 3 AZ，基于 Ephemeral NVMe，write-through 并由对象存储回填）；元数据统一由 ZooKeeper 管理，持久层为 Object Store（如 S3）。

### Figure 3: SalesforceDB Multi-tenancy

![[SalesforceDB_fig3_multitenancy.png]]

**说明**：对比传统 DBMS（表内租户数据混合排列）与 SalesforceDB（同表内租户数据各自聚簇连续存放），直观展示 [[Data Co-location|租户数据聚簇]]带来的扫描效率优势。

### Figure 4: Virtual References

![[SalesforceDB_fig4_virtual_references.png]]

**说明**：跨 LSM 多层（$L_0, L_1, \dots, L_7$）的 Extent Reference（ER1~ER11）中，与 Tenant B 键区间相交的部分被复制、裁剪为虚拟引用（绿色部分为 Tenant B 在 LSM 中的键区间），组合即构成 Tenant B 的快照。是 [[LSM Virtualization]] 机制的核心图示。

### Figure 5: SalesforceDB Scale-out

![[SalesforceDB_fig5_scaleout.png]]

**说明**：主读写集群由单个 Coordinator 节点与多个 Executor 节点构成；Executor 通过 RPC 与 Coordinator 交互，并各自 tail Coordinator 经 SDBStore 持久化的 Redo Log；图例标注了 Active / Partially active / Inactive 三种 Memstore/Redo Log 状态。

### Figure 6: Reads at Executor Nodes

![[SalesforceDB_fig6_reads_executor.png]]

**说明**：以 XCN 时间轴展示 Executor 与 Coordinator 的 Memstore 覆盖范围差异：Coordinator 维护从 Commit Epoch 到 Persistence XCN 的完整 Memstore；Executor 的本地 Memstore 覆盖 Redo XCN 到 Persistence XCN 之间的 "Read RPC Range"，低于 Persistence XCN 的数据经 SDBStore 持久化后本地可直接访问。

### Figure 7: SalesforceDB Scale-out（吞吐与延迟）

![[SalesforceDB_fig7_scaleout_throughput.png]]

**说明**：节点数从 1 扩展到 16 时，相对单节点的吞吐提升达 **11.79×**，而平均请求延迟仅从约 1.84ms 增至约 2.01ms（约 15% 增幅），体现近线性扩展同时延迟稳定。

### Figure 8: CPU Utilization — Coordinator vs. Executor

![[SalesforceDB_fig8_cpu_utilization.png]]

**说明**：节点数 ≤5 时 Coordinator 同时处理写协调与 SQL 处理，CPU 占用略高于 Executor；6 节点及以上 Coordinator 仅处理写协调，CPU 占用显著下降，为继续扩展留出空间，解释了为何 5 节点与 6 节点集群吞吐相近。

### Table: 关键生产指标汇总（原文为叙述性数据，整理为表）

| 指标类别 | 数值 |
|---|---|
| 生产可用性（30 秒以上不可写视为不可用） | 99.99% |
| 全机群日处理 SQL 语句数 | > 4.8 万亿条/天 |
| 全机群日记录变更数（增/改/删） | > 1 万亿条/天 |
| 单个 AZ 完全不可用的重大相关性故障事件（2022 年以来） | 6 次，均未影响客户 |
| 物理损坏事件（5 年以上生产部署） | 0 次 |
| 强制数据库故障转移平均耗时 | 7.91 秒（p95: 11 秒） |
| SDBStore group commit 延迟 | 平均 3.03ms（p95: 3.74ms） |
| SDBStore 日志写延迟 | 平均 2.32ms（p95: 3.60ms） |
| SDBStore 随机数据读延迟 | 平均 0.49ms（p95: 1.20ms） |

**说明**：这些来自 2026 年 4–6 月的生产数据，印证了论文设定的高可用、低延迟目标在大规模生产环境中达成。

### Table: LSM 虚拟化效率（2026 年 4–6 月生产数据）

| 场景 | 关键指标 |
|---|---|
| 沙箱克隆（≥10TB 租户） | 相比逐行拷贝提速 **2.5×**；85+TB 超大租户提速 **7.3×**（22 天 → 3 天） |
| 沙箱创建总量 | 89,163 个（含 584 个超 1TB），平均创建时间从 8.02 小时降至 5.51 小时（**-31%**） |
| LSM 虚拟化步骤本身耗时 | 平均 84.4 秒（p95: 279.9 秒），仅占端到端沙箱创建时间 p95 的 **0.69%** |
| 零拷贝租户迁移 | 6,596 个租户迁移，平均耗时 63 秒（p95: 151 秒） |
| 租户级加密 | 7,000+ 租户加密，单 fragment 加密开销 13.96μs（p95: 49.58μs，相对 2.7ms 总写入时间可忽略），解密开销 12.52μs（p95: 25.77μs，相对 0.49ms 总读时间可忽略） |

**说明**：定量验证了 [[LSM Virtualization]] 带来的沙箱创建加速、零拷贝迁移效率及加密开销可忽略不计这三项核心收益。

### Table: 协调最小化效果（16 节点集群实测）

| 指标 | 数值 |
|---|---|
| 触发 RPC 的 fetch 占比 | 0.88% |
| 触发 RPC 的 scan 占比 | 3.23% |
| Memstore Bloom Filter 消除的 fetch RPC | 99.22% |
| Memstore Bloom Filter 消除的 scan RPC | 71.06% |
| 需要停顿以保证一致性的提交占比 | 0.67%，平均停顿 560μs（p95: 990μs） |

**说明**：量化验证了 [[Bloom Filter]] 等协调最小化机制在真实生产集群中的有效性，以及外部一致性机制带来的性能开销极低。

---

## 实验

### 评估场景

论文评估分为两部分：（1）2026 年 4–6 月的生产机群观测数据（可用性、存储延迟、LSM 虚拟化效率、协调最小化效果）；（2）针对计算横向扩展的合成基准实验。

### 合成负载与基准设置（Section 7.4）

- 负载包含 6 张多租户表、4 个只读 + 4 个读写业务事务，读写比 70%/30%；查询涵盖点查到数十万行的范围扫描，以及返回最多 500 行的 Join；数据访问偏向近期写入（skewed towards recent writes）。
- 运行环境：AWS `r6i.32xlarge` 实例，计算节点从 1 扩展到 16；每新增一个计算节点，数据集新增 10 个大租户 + 100 个小租户共享多租户表，大租户的规模和请求率是小租户的 5 倍。
- 每种配置均在查询处理节点 60% CPU 利用率下运行，请求率通过插值归一化到各配置的 60% CPU。

### 关键实验结果

- 16 节点相对单节点达到 **11.79× 容量增益**（同等利用率下的吞吐），日志记录写入吞吐达 **516K 条/秒**（55MB/秒），事务提交 **17,968 次/秒**；提交延迟从 1 节点的 1.84ms 仅增至 16 节点的 2.01ms；16 节点时系统处理 **677K RPC/秒**。
- 高倾斜测试（引入对同一键大量写入的第 9 个业务事务）：吞吐仍可扩展 **12.63×**（1→16 节点），但 Coordinator CPU 利用率在 16 节点时超过 92%，成为扩展瓶颈，平均请求延迟增加 48%——这是当前单 Coordinator 架构的已知扩展上限，论文在"经验教训"中提到下一代架构将对 Coordinator 职责做进一步解耦（disaggregating）。
- 生产案例：某大客户实例随节假日流量从 5 扩展到 9 再到 16 节点，归一化到 30% CPU 利用率，查询执行速率从 515K/秒（5 节点）增至 874K/秒（9 节点）、1,519K/秒（16 节点），体现近线性扩展。

### 可视化结果

Figure 7、Figure 8（见上文"关键图表"）分别展示吞吐/延迟扩展曲线和 Coordinator/Executor CPU 利用率对比，是本节的核心数据支撑。

---

## 批判性思考

### 优点

1. **架构思想高度统一且自洽**：把"不可变性"作为贯穿存储、缓存、协调最小化、租户虚拟化、加密等几乎所有子系统的统一设计原则，论文第 9 节"Immutability changes everything"清晰总结了这种跨层杠杆效应，是工程设计上值得借鉴的范式。
2. **生产数据充分、量化具体**：评估部分大量使用真实生产机群（数千实例、400 万+租户、4.8 万亿 SQL/天）的实测数据而非仅实验室 benchmark，可信度和说服力强，尤其是 LSM 虚拟化在沙箱创建、迁移、加密上的收益都给出了具体百分比和绝对时间。
3. **"元数据操作替代物理拷贝"的设计具有普适借鉴价值**：与 [[Snowflake]] 的零拷贝表/库克隆思路一脉相承，但 SalesforceDB 更进一步把租户身份虚拟化嵌入存储引擎本身（而非仅在元数据层操纵表/库级快照），并支持跨数据库实例迁移，这是相对 Snowflake、FoundationDB Record Layer 等系统的差异化贡献。
4. **协调最小化机制设计精巧**：单 Coordinator + 多 Executor 的非对称架构，结合 Memstore Bloom Filter、批量/异步 RPC、本地 Memstore 维护未提交写等多种手段，在不引入分区或专用硬件的前提下做到近线性扩展，工程细节丰富可操作性强。

### 局限性

1. **单 Coordinator 是明确的扩展瓶颈**：论文自己在高倾斜测试和"经验教训"部分都承认，单一 Coordinator 节点在极端写热点场景下 CPU 利用率可达 92%+，是当前架构的硬限制；论文没有给出具体的下一代"职责解耦"方案细节，只是预告。
2. **性能隔离能力弱于专用隔离方案**：论文自己也承认 shared-table 多租户比其他模型"提供更低程度的性能隔离"，依赖应用层限流而非数据库内生的资源配额机制（如 SQLVM 的 per-tenant quota），在噪声邻居场景下的硬隔离保证不如这些方案强。
3. **缺乏形式化的正确性证明**：对于外部一致性、租户翻译、跨数据库 extent 删除时序等关键正确性机制，论文仅做了文字描述和生产验证，没有给出协议级别的形式化证明或 TLA+ 之类的模型检验，不如学术系统论文（如 Spanner）严谨。
4. **评估的 benchmark 场景相对单一**：合成负载虽然力图模拟生产流量特征，但只有 6 张表、固定的大/小租户比例结构，难以反映 Salesforce 真实应用 schema 的复杂度（例如几十万张表级别的 schema 规模），工业界论文常见的"简化基准"局限性在这里同样存在。

### 潜在改进方向

1. 对 Coordinator 单点的解耦（如按 keyspace 或租户维度做多 Coordinator 分片并处理跨 Coordinator 事务）是论文明确提到的下一步工作方向，值得持续关注其后续论文或系统演进。
2. 可以探索更细粒度的、数据库内生的性能隔离机制（类似 SQLVM 的配额体系）与现有基于应用层限流的方案结合，以降低对应用层观测/限流逻辑的依赖。
3. LSM 虚拟化依赖 schema fingerprint 做跨库兼容性校验，未来可考虑更细粒度的 schema diff 机制，支持部分不兼容场景下的"降级迁移"（如自动跳过目标库多出的索引）而非要求完全一致。

### 可复现性评估

- [ ] 代码开源（SalesforceDB/SDBStore 均为 Salesforce 内部专有系统，未开源）
- [ ] 预训练模型（不适用）
- [x] 训练细节完整（系统架构与评估方法描述详尽，合成 benchmark 的负载构造、硬件型号、CPU 利用率归一化方法均有说明）
- [ ] 数据集可获取（生产数据为 Salesforce 内部机群观测，合成 benchmark 未开源数据集或代码）

---

## 关联笔记

### 基于
- [[PostgreSQL]]：SalesforceDB 的 SQL 层基础，复用其查询处理、事务处理、schema 管理能力，但替换了存储与访问方法层。
- [[Apache BookKeeper]]：SDBStore 的底层基础，提供 quorum 写入、fencing 机制与非 POSIX API。

### 对比
- [[Snowflake]]：同样通过操纵不可变共享存储上的元数据实现零拷贝克隆，但仅在表/库粒度；SalesforceDB 进一步将租户身份虚拟化嵌入 LSM 存储引擎，支持跨数据库实例迁移。
- [[Spanner]]：同为追求外部一致性的分布式数据库，但 Spanner 依赖高精度时钟服务（TrueTime）和基于分区的横向扩展；SalesforceDB 用单时间戳 oracle + 混合逻辑时钟，且不依赖分区。
- [[CockroachDB]]、[[CockroachDB Serverless]]：同样做计算存储解耦和多租户，但走范围分区路线（需要跨分区两阶段提交），且 Serverless 版本租户间不共享 schema，难以做到统一 schema 下的跨库迁移。
- [[FoundationDB Record Layer]]：同样按租户聚簇数据，但是在存储层之上的逻辑租户作用域子空间实现，而非像 SalesforceDB 那样把多租户嵌入存储引擎本身。
- [[Amazon Aurora]]、[[PolarDB]]：同属计算存储分离的云原生 OLTP 数据库，但写请求仍必须由单一节点服务（Aurora 靠只读副本扩展读），而 SalesforceDB 要求多节点都能服务读写以支撑其最大租户的工作负载。

### 方法相关
- [[LSM-tree]]：SalesforceDB 存储引擎的基础数据结构，本文强调其不可变性带来的跨系统收益。
- [[Bloom Filter]]：既用于 LSM 层减少读放大，也用于 Executor-Coordinator 间减少 RPC（Memstore Bloom Filter），是"过滤是跨层架构模式"这一经验教训的具体体现。
- [[MVCC]]：SalesforceDB 读写路径的并发控制基础，结合 lineage 校验防止逻辑损坏。
- [[Two-Phase Commit]]：SalesforceDB 明确避免依赖的协调机制，通过不分区+单 Coordinator 架构规避。

### 硬件/数据相关
- [[AWS S3]]、[[AWS EBS]]：SDBStore 数据缓存和日志缓存分别依赖的云存储基础设施。
- [[ZooKeeper]]：协调层基础，提供领导者选举、quorum 服务与元数据存储（BookKeeper 文件到节点映射）。

---

## 速查卡片

> [!summary] Salesforce Database: A Cloud-Native Multi-Tenant OLTP Database
> - **核心**: 把租户聚簇嵌入单一不可变 LSM 存储引擎，用"LSM 虚拟化"把租户迁移/克隆/加密变成元数据级操作
> - **方法**: shared-table 多租户 + galactic keyspace 聚簇 + Virtual File References（ER/ERS）+ 单 Coordinator/多 Executor 低协调扩展
> - **结果**: 生产可用性 99.99%，16 节点吞吐 11.79× 扩展且延迟仅增 15%，大租户沙箱克隆提速最高 7.3×（22天→3天），超 7000 租户加密开销可忽略
> - **代码**: 未开源（Salesforce 内部专有系统）

---

*笔记创建时间: 2026-10-10*
