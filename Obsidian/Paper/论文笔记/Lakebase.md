---
title: "Lakebase: Serverless Postgres over Open Lake Storage"
method_name: "Lakebase"
authors: [Jasraj Dange, Andrei Dragus, Ali Ghodsi, Haoyu Huang, Yihe Huang, Stas Kelvich, Heikki Linnakangas, Hans Norheim, Ippokratis Pandis, Nikita Shamgunov, Em Sharnoff, John Spray, Zhou Sun, Reynold Xin, Matei Zaharia]
year: 2026
venue: PVLDB 19(12)
tags: [cloud-native-database, serverless, storage-compute-separation, lakehouse, database-branching, object-storage, postgresql, autoscaling]
zotero_collection: VLDB/2026
image_source: local
created: 2026-09-27
---

# 论文笔记：Lakebase: Serverless Postgres over Open Lake Storage

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | Databricks（吸收 [[Neon]] 团队与开源社区） |
| 日期 | 2026，PVLDB 19(12): 4385–4398 |
| 项目主页 | 无独立论文主页；产品页 Databricks Lakebase |
| 对比基线 | [[Amazon Aurora]]（Gen-2 代表）、Gen-1（多 AZ 复制盘上的 PostgreSQL） |
| 链接 | [DOI](https://doi.org/10.14778/3827998.3828040) / [代码](https://github.com/neondatabase/neon)（基于开源 Neon） |
| 原始 PDF | `VLDB/2026/Lakebase- Serverless Postgres over Open Lake Storage.pdf` |

---

## 一句话总结

> Lakebase 把 PostgreSQL 的全部数据（含 [[WAL]]）以**开放 PostgreSQL 页格式**落在云对象存储上，用 [[Safekeeper]] 做 WAL 共识复制、[[Pageserver]] 做页物化，从而同时拿到秒级冷启动、$O(1)$ 分支、scale-to-zero 与多引擎共享存储。

---

## 核心贡献

1. **第三代数据库架构**：区别于 Gen-1（一体机，如 MySQL/PostgreSQL/Oracle）和 Gen-2（厂商内部拆存储但格式私有、单引擎访问，如 Aurora/Socrates/AlloyDB），Lakebase 把存储外部化为**开放格式的 lake 层**，[[PostgreSQL]] 计算引擎在其上无状态地弹性伸缩。
2. **开放格式 + 多引擎访问**：所有持久化数据是标准 PostgreSQL 页 + 开放湖格式，[[Lakehouse]] 的 Spark / Lakehouse//RT 可绕过 PostgreSQL 计算直接读同一份物理数据，实现零拷贝 HTAP。
3. **面向 Agent 的数据库原语**：[[Database Branching|零拷贝分支]]（$O(1)$ 元数据操作）、亚秒级冷启动（warm pool）、[[Scale-to-Zero]]、自动跨 AZ 故障切换。生产观测每天创建 1000 万+ 数据库，Agent 创建量是人类用户的约 4 倍，分支深度有超过 500 的层级。
4. **可预测的存储格式**：[[Pageserver]] 内用**二维**（page key × LSN）[[Layer File|LSM 式]]存储格式，写放大 $O(1)$、读代价 $O(I)$、空间放大 $O(1+R+wP/N)$，生产实测约 7× 空间放大。

---

## 问题背景

### 要解决的问题

生产数据库架构仍停留在 1980 年代的设计假设（存储稀缺、网络慢、无云弹性），与现代基础设施错位，表现为三个持续痛点：

1. **计算存储紧耦合** → 按峰值预置容量、常态闲置；负载超预置后性能陡降；快照/回填/合规清理与生产流量争抢资源；数据库成为基础设施中最脆弱的组件。
2. **与当代研发流程错位** → 代码可瞬时分支，但"起一个真实数据库环境"要几分钟到几小时，数据库成为迭代速度的瓶颈。
3. **结构性厂商锁定** → 存储格式与单一引擎绑定，迁移昂贵，分析集成要靠 ETL 复制数据，跨云可移植性受限。

### 现有方法的局限

Gen-2 系统（Aurora [[Socrates]] AlloyDB SolarDB Taurus Tell）虽然在厂商内部做了计算存储分离，改善了弹性与可用性，但**分离停留在厂商架构内部**：格式仍私有、数据仍只能通过主引擎访问。Snowflake Unistore 用私有行存给数仓加 OLTP；Microsoft Fabric 把数据以 Delta 表放 OneLake，但 OLTP 引擎格式仍私有。结果是"过渡性架构"——性能改善，但可移植性、开放性、多引擎访问没有根本解决。

### 本文的动机

Agent 化工作负载从三个维度重塑数据库设计，而这三个维度恰好都指向"开放存储 + 弹性计算"：

- **成本敏感与应用泛滥**：Agent 让应用创建成本骤降，产生大量短命、常闲置、单独价值低的实验性服务。生产遥测（Figure 1）显示**大量实例存活不足 10 秒**。传统"长活实例 + 持续预置计算"的假设完全失配。
- **演化式工作流**：应用可在数分钟内从零生成，迭代时**代码和数据库一起分支**（Figure 2 显示许多项目有深层分支，部分超过 500 个分支），checkpoint/branch 必须成为一等原语，即数据库版的 `git checkout -b`。
- **开放生态**：Agent 在公共语料中充分代表的开放生态上表现最好，强烈偏好开源接口与格式，尤其 PostgreSQL 兼容；这要求存储层也是开放的，让外部工具、Agent、分析引擎无需私有基础设施即可读写。

---

## 方法详解

### 总体架构

Lakebase 采用 **[[Disaggregated Storage-Compute Architecture|计算存储完全分离]]** 架构（Figure 3）：

- **控制面**：Lakebase Manager 管理 PostgreSQL 计算节点池；Storage Controller（SC）管理多租户存储（[[Pageserver]] PS + [[Safekeeper]] SK）。
- **数据面 — 计算**：PostgreSQL 计算节点（primary + 读副本）跑在裸金属 + 本地 SSD 上；写入路径经 WAL proposer 广播到 SK，读取路径经 SMGR 访问 PS。
- **数据面 — 存储**：SK 负责 WAL 的共识复制与上传；PS 负责把 WAL 物化成页并按 LSN 提供服务。
- **持久层**：所有数据（含 WAL）最终落在云对象存储，格式开放。
- **多引擎**：Lakehouse 侧 Spark / Lakehouse//RT 可 Direct read，亦可 reverse-ETL 写入；PostgreSQL 侧通过 ETL（wal2delta）前向同步。

### 设计目标（6 条）

1. **利用高持久对象存储**：AWS S3 提供 11 个 9 的持久性、跨 AZ 复制、按访问模式自动分层、版本化与恢复能力，成本远低于自营盘。
2. **利用云基础设施做高可用与性能**：OLTP 需要毫秒级响应，对象存储单独做不到，因此引入**存储层级**：计算本地 SSD 缓存 → PS 物化并缓存页 → 冷键回源对象存储。
3. **计算快速启动**：冷启动是 agentic 负载的关键路径（每次用户 prompt 都可能起一个新计算）。用 [[Warm Pool]] + 把 crash recovery 卸载到分布式存储，把启动压到亚秒。
4. **零拷贝分支**：分支是 $O(1)$ 元数据操作，使"数据也随代码一起分支"在 prompt 级延迟下可行。
5. **计算弹性与 scale-to-zero**：监控计算资源用量与工作集大小，及时升配、降配、闲置归零。
6. **运维简化**：自动伸缩、分支与 PITR 替代备份、自动跨 AZ 故障切换。

### 组件详解

#### 组件 1: PostgreSQL Compute

- 每实例一个 primary + 可选读副本；primary 复制到 **3 个 SK**，初始挂 1 个 PS，随库增长分片到更多 PS。
- 节点运行 PostgreSQL + **定制 storage manager**，实现 PostgreSQL SMGR API，对接 PS 并更新本地 SSD 缓存。
  - SMGR **读**：命中本地缓存直接返回，miss 则从 PS 取回并缓存。
  - SMGR **写**：只更新本地缓存——因为 PostgreSQL 恢复只需已持久化的 WAL。
- **WAL proposer**（PostgreSQL 扩展）：持续把 WAL 记录广播给所有 SK，达到 quorum 持久后才推进。
- **启动路径**：向 PS 请求 basebackup（启动所需元数据文件），**避免 checkpoint**；恢复点取 PS 已摄入的最后一条 WAL 记录，而不是最后一个 checkpoint。
- **读扩展**：读副本共享 primary 的 SK，持续拉 WAL 更新本地缓存；另有 **static replica** 钉在某个 [[LSN]] 上用于时间旅行查询。

#### 组件 2: Safekeeper（SK）

- 维护一组隔离的 [[Timeline]]，每个 timeline 存 WAL 段；本地盘保留最新 WAL，并把 quorum 已提交的段主动上传对象存储。
- **共识复制**：类 [[Raft]] 协议，PostgreSQL primary 是 WAL proposer（leader），SK 是 acceptor；一条 WAL 记录在 quorum 写盘后才算持久；落后的 SK 副本由最先进的副本回填。
- **防脑裂**：计算必须先自选为 leader 才能写；SK 维护单调递增的 **term**，新 primary 递增 term 并向 SK quorum 自我提名，SK 只接受见过的最高 term；一旦检测到 term 变化，计算必须重新竞选。相关概念见 [[Split-Brain]]、[[Leader Election]]。

#### 组件 3: Pageserver（PS）

- 从 SK 拉 WAL，维护 tenant/timeline，对计算暴露两个 API：`GetPage@LSN` 与 `GetBaseBackup`。
- 把 delta WAL 记录与物化整页镜像组织成 **LSM-tree 式**结构（见下节）。
- `GetPage@LSN` 调用时：找到更早 LSN 的最后一个 full-page image，把 delta WAL 记录重放到目标 LSN，返回该页；**请求会阻塞直到 PS 摄入到该 LSN**。
- 定期把树的快照与元数据上传对象存储以加速崩溃恢复；每个 timeline 保有 **hot standby**，预先下载快照与热层文件，primary 故障时立即接管并从已缓存层服务。

#### 组件 4: Storage Controller（SC）

- 统一管理 SK 与 PS：按 **AZ 亲和**把新 timeline 分配到最闲的 SK/PS，并在后台随负载演化重新优化布局。
- 通过周期性 liveness ping 监控健康：SK 故障 → 快速把受影响 timeline 迁到新副本恢复冗余；PS 故障 → 通知 hot standby 接管。
- 库变大时**透明且零停机地**把 timeline 分片到多个 PS，提升 GetPage 吞吐；每个分片从某个 SK 副本摄入自己相关的 WAL 记录。

#### 组件 5: 非 PostgreSQL Compute

外部引擎可直接操作共享存储，获得 Lakebase 与 Lakehouse 数据的统一视图：

- **写入侧**：Lakehouse 表直接同步进 Lakebase 存储，绕过 PostgreSQL 计算；Spark 等分布式执行器生成 PostgreSQL 兼容页文件写入对象存储，PostgreSQL 随后当作普通表消费。
- **分析侧**：Lakehouse//RT 等查询引擎直接读对象存储中不可变的 PostgreSQL 页文件；查询绑定到一个已提交快照，运行在一致视图上但绕过 PostgreSQL 计算 → 零拷贝访问活数据、不干扰事务负载、可水平扩展并做向量化处理。
- 这种"共享存储上的多引擎访问"正是第三代与第二代的分水岭：第二代只通过主引擎暴露数据。

---

## 关键公式

### 公式1: [[Write Amplification|写放大]]（常数级）

$$
WA = O(1) = 1 + O\!\left(\frac{1}{I}\right) + O\!\left(\frac{1}{R}\right)
$$

**含义**: PS 存储格式的写放大是常数。每个字节在 delta 摄入时写一次；镜像创建贡献 $O(1/I)$；GC-compaction 贡献 $O(1/R)$。

**符号说明**:
- $I$: **image threshold**，某个 key range 上允许堆叠的最大 delta 层数，超过即写新 [[Layer File|image layer]]；它约束单次读的重建深度。
- $R$: **GC-compaction 触发比**，新累积历史达到上一次 compact 后 base 大小的 $R$ 倍时触发 GC-compaction，把 GC horizon 以下的全部历史塌缩成一个新 base。
- $N$: 最新 LSN 上的活跃数据集大小（以页计）。
- $w$: WAL 写入（摄入）速率。
- $P$: [[PITR|point-in-time recovery]] 窗口，超出该窗口的历史可回收。

> 与经典 LSM 的最大差别：**层数是常数**。作者明确表示不声称该格式是 state-of-the-art，而是为**可预测性**调的。

### 公式2: [[GetPage@LSN]] 读代价

$$
Cost(\text{GetPage@LSN}) = O(I)
$$

**含义**: 读只支持点查（无 range-scan 原语），代价为 $O(I)$ 次层访问，因为 image threshold 限定了需要重放的 delta 链长度。

**符号说明**:
- $I$: 同上，image threshold，限定 delta 链长度。
- 实测计算侧缓存命中率通常 **> 98.5%**，掩盖了大部分 GetPage 延迟。

### 公式3: 空间放大

$$
SpaceAmplification = O\!\left(1 + R + \frac{w \cdot P}{N}\right)
$$

**含义**: Lakebase 把空间放大定义为 **(PS 总存储大小) / (用户在最新 LSN 上的逻辑数据库大小)**。三项分别是：活状态的一份物化（$1$）、尚未回收的 GC slack（$R$）、窗口内保留的历史（$wP/N$，通常主导）。

**符号说明**:
- $R$, $w$, $P$, $N$: 同上。
- 该比值天然是**高估**，因为系统有义务保留 PITR 历史且对该历史计费。生产舰队实测约 **7×**。

### 公式4: 分支语义（分支点约束）

$$
\text{BranchLSN} \in \text{PITR-Window}(\text{parent}), \qquad \text{diverge}(t) = \begin{cases} \text{shared}, & t \le \text{BranchLSN} \\ \text{independent}, & t > \text{BranchLSN} \end{cases}
$$

**含义**: 分支时 [[LSN|branch LSN]] 必须保证"在分支创建的那一刻，父分支上该 LSN 处的每一个页都可被重建"。为简化，实现把 branch LSN 限制在父分支的 PITR 窗口内。分支前共享历史与**同一份物理存储**，分支后各自独立追加 WAL。（？？？

**符号说明**:
- $\text{BranchLSN}$: 父子分支的分叉点（Figure 4 中编号 ⑪）。
- 未在 branch LSN 提交的事务，按崩溃恢复中的"在途事务"处理：**中止**，其待定修改在子分支不可见。

### 公式5: 写入持久性条件（quorum）

$$
\text{durable}(w) \iff \left|\{\, s \in \text{SK} : \text{flushed}(s, w) \,\right\}| \ge \left\lfloor \frac{|\text{SK}|}{2} \right\rfloor + 1
$$

**含义**: 一条 WAL 记录只有在 SK **quorum** 在当前 term 下刷盘后才被确认；proposer 的 commit LSN 取 quorum flush LSN。这解释了为什么 Lakebase 只有"单路 quorum 复制写"——Gen-2 把存储扇出消耗在计算核上，而 Lakebase 不这么做。

**符号说明**:
- $s$: 单个 [[Safekeeper]] 实例。
- $\text{flushed}(s,w)$: SK $s$ 已将记录 $w$ 刷入本地 segment 文件（flush LSN 覆盖 $w$）。
- 部署上 $|\text{SK}| = 3$ 且分布在 3 个 AZ，容忍单节点或单 AZ 故障。

### 公式6: 内存伸缩约束（两条水位线）

$$
\text{allocated} + \text{pagecache}_{\le \text{desired}} < 0.90 \cdot M_{total}
\qquad\text{且}\qquad
\text{allocated} < 0.75 \cdot M_{total}
$$

**含义**: 内存目标尺寸同时满足两个约束：常态下让计算缓存占用绝大多数内存（90% 水位），分配密集场景下额外保证纯分配内存不超过 75%（为缓存预留等量增长空间）。

**符号说明**:
- $\text{allocated}$: VM 内已分配内存。
- $\text{pagecache}_{\le \text{desired}}$: 处于目标缓存大小范围内的页缓存内存。
- $M_{total}$: 分配给虚拟机的总内存。

---

## 关键图表

### Figure 1: Compute lifetime observed in production（生产环境计算生命周期分布）

![[Lakebase_fig1_compute_lifetime.png]]

**说明**: 取自 Databricks Lakebase 服务的真实遥测。横轴为计算存活时长分桶，纵轴为计算占比。**1–10 秒区间占比最高（约 55–58%）**，而 1+ months 的长寿实例仅约 10%。这直接否定了传统 OLTP"长活实例 + 持续预置算力"的部署假设，是 [[Scale-to-Zero]] 与亚秒启动设计的实证依据。

### Figure 2: Branch depth observed in production（生产环境分支深度分布）

![[Lakebase_fig2_branch_depth.png]]

**说明**: 横轴为项目分支深度分桶，纵轴为项目占比。**深度为 2 的项目占比最高（约 63%）**，随后快速衰减（[3-4] 约 20%，[5-8] 约 8%，[9-16] 约 5%，[17-64] 约 3%，[64+] 约 1%）。虽然长尾占比小，但正文指出实际存在**超过 500 个分支**的深层层级，说明分支原语不能只做"少量分支"的优化，必须支持深树且读路径能容忍跨 timeline 递归。

### Figure 3: Lakebase architecture（Lakebase 总体架构）

![[Lakebase_fig3_architecture.png]]

**说明**: 完整架构图。**计算侧**（虚线之上）：Proxy 承接入站连接，[[PostgreSQL]] 计算节点池由 Lakebase Manager 管理。**存储侧**（虚线之下，Multi-tenant Storage）：Safekeeper 组（3 副本）与 Pageserver 组由 Storage Controller 管理，两组件间流动 WAL streams。**持久层**：底部绿色为 Cloud Object Storage（图中标出 Parquet、Delta、Iceberg 等开放格式标记）。**右侧**：Lakehouse 通过 Direct read / ETL / Reverse ETL 三条通路与共享存储交互。注意 WAL 与页数据都落在对象存储，这是"开放"的关键。

### Figure 4: WAL and LSNs maintained by components（各组件的 LSN 体系）

![[Lakebase_fig4_wal_lsns.png]]

**说明**: 把整个系统的 [[LSN]] 语义画成一张图，是理解 Lakebase 正确性模型的核心。上半为 **Database Log**（父/当前分支日志），下半为 **Branch Log**（子分支日志）。

- **Compute LSNs**：① Current LSN（追加点）② Commit LSN（quorum 持久点，崩溃恢复恢复到此处）③ Replica replay LSN（副本可见点）④ Static replica LSN（钉住的读点）。
- **SK LSNs**：⑤ Flush LSN（刷盘点）⑥ Commit LSN（由 primary 通知，SK 只向其下的数据流给 PS）⑦ Backup LSN（已上传对象存储、可删本地段的水位）。
- **PS LSNs**：⑧ Last record LSN（内存摄入点）⑨ Flush LSN（批量落盘点，崩溃从此再水化）⑩ Remote consistent LSN（已上传远端，本地无有效 flush LSN 时的恢复来源）⑪ Branch LSN（分叉点，位于 Branch Log 与 Database Log 之间）。

### Figure 5: Pageserver storage organization（Pageserver 存储组织）

![[Lakebase_fig5_pageserver_storage.png]]

**说明**: 二维（Key × LSN）层文件布局。绿色矩形 = **delta layer**（一段 key 范围 × 一段 LSN 范围的原始 WAL 记录），黑色粗线 = **image layer**（某一 LSN 上的一段 key 范围的整页快照）。一次 `GetPage(key=k, lsn=l)` 分两步：**① 在 LSN < l 的层中找到 k 的最新镜像**；**② 找到自该镜像 LSN 到 l 之间所有触及 k 的 WAL 记录并重放**。这正是 [[LSM-tree]] 思想在"页版本"维度上的应用，与按 key 单维组织的经典 KV LSM 不同。

### Figure 6: Read path – GetPage@LSN（读路径）

![[Lakebase_fig6_read_path.png]]

**说明**: 阶段 ① 命中 LFC（[[Local File Cache]]）直接返回页；② 维护 LwLSN 缓存（页 → not-modified-since LSN 的固定大小映射，脏页淘汰时更新）；③ 未命中则发 `GetPage@LSN` 给 [[Pageserver]]，请求携带两个 LSN（request LSN 指定要取的页版本，not-modified-since LSN 承诺该页在两者之间未变，使 PS 只需摄入到后者即可，避免因等待追平 current LSN 而阻塞）；④⑤ PS 可能需从对象存储拉取层文件。图示右侧即 Figure 5 的二维结构。预取读走同一路径。

### Figure 7: Write path（写路径）

![[Lakebase_fig7_write_path.png]]

**说明**: ① WAL proposer 在启动时提出新 term 并自选为 leader，SK quorum 确认更高 term 后开始追加 WAL；② 每个 SK 把 WAL 记录追加到本地 segment 文件并周期性刷盘，返回自己的 flush LSN 与已知 commit LSN；③ 3 个 SK gossip commit LSN 并确定性地选出一个 upload leader（跨 timeline 做负载均衡）；④ SK 主动把 WAL 段上传对象存储；⑤ PS 从 SK 拉 WAL 摄入，SK 会解码每条记录影响的 page id 并**只路由到相关的 Pageserver**；元数据（如 relation size）只由特殊的 **Shard-0** 跟踪，故所有元数据更新都发往 Shard-0。当写入快于 PS 摄入时 proposer 施加**背压**。

### Figure 8: High availability（高可用拓扑）

![[Lakebase_fig8_high_availability.png]]

**说明**: 跨 3 个 AZ 的部署。计算 primary 与副本跨 AZ 分布，故障节点在 **500 ms 内**用新节点修复（warm pool + 存储解耦）；用户可配置带 hot standby 的计算 HA 组。写复制到 3 个 SK（位于 3 个 AZ），容忍一个节点或 AZ 故障。每个 PS 分片在**不同 AZ** 保有 hot standby，主动下载主副本快照中最近使用的层文件，primary 失败时立即接管并从已缓存热数据服务 GetPage。故障切换全自动、无需人工介入。

### Figure 9: P95 compute startup time（P95 计算启动时间）

![[Lakebase_fig9_compute_startup.png]]

**说明**: 纵轴为对数刻度（ms），横轴为时间（小时，跨度约 700+ 小时）。**蓝色（Lakebase 带 warm pool）稳定低于 500 ms**；**红色（无 warm pool）落在 10,000–50,000 ms 量级**，且出现远超 50,000 ms 的尖峰。正文补充：无 warm pool 时启动新计算需要数分钟，**尾延迟可达数小时**（云厂商 VM 供给偶发极慢）。启动时间四个阶段拆解：请求 basebackup ~100 ms → 同步 Safekeeper ~5 ms（同时声明唯一 primary 防脑裂）→ 启动计算实例 150 ms → 配置 < 5 ms。

### Figure 10: Compute autoscaling（计算自动伸缩）

![[Lakebase_fig10_autoscaling.png]]

**说明**: 取自生产股票交易应用的日周期负载回放。左轴 QPS，右轴 CPU 数。计算配置 min 1 CPU / max 7 CPU。初期 1 CPU；负载上升时伸缩 agent 增加 3 个 CPU。**第 44 分钟开盘，查询量从 32 QPS 骤增至 10K QPS**，估计工作集超过当前 LFC 大小，LFC 命中率跌破 30%——agent 立即把 CPU 拉到上限。白天 CPU 稳定在约 6，随波动在 7 与 5 间调整。全程 **Actual QPS 几乎贴合 Target QPS**，证明伸缩不干扰客户负载。闲置满 5 分钟（默认）后计算关闭。与 7-CPU 固定规格对比：自动伸缩实例响应时间仅略高（固定实例前期 LFC 更大、miss 更少），稳定后延迟相当，但**自动伸缩整体 CPU-hours 更少**。

### Figure 11: TPC-H of Lakebase vs Lakehouse//RT w/ Direct Access（TPC-H 对比）

![[Lakebase_fig11_tpch_direct_access.png]]

**说明**: TPC-H scale factor 10，Lakehouse//RT + Direct Access 使用**与 Lakebase 相同的硬件资源**，故差异纯来自执行引擎而非资源。纵轴为时间（秒，对数刻度）。整体上 **Direct Access 把总执行时间降低 10×，几何平均查询延迟改善 3×**。分查询看：Q1 提速 11×（并行聚合更高效，PostgreSQL 并行用多进程、Direct Access 用线程）；**Q18 接近 90×**；Q17 提速 19×（把子查询改写成 join 计划，而 PostgreSQL 反复求值子计划）。**反例**：Q8 与 Q19 更慢，因为当前实现只支持顺序扫描，无法使用索引嵌套循环连接。此外在 400 QPS 并发 OLTP 负载下，Direct Access 扫描因缓存失效慢 1.76×，但因为跑在计算之外，**从不影响 primary 的写延迟**。

### Table 1: Peak HammerDB TPROC-C NOPM（x1000）

| Tier (vCPU) | Gen-1 | Gen-2 | Lakebase |
|---|---|---|---|
| 1 | — | 9.0 (1.0×) | **23.3 (2.6×)** |
| 4 | 92.6 (1.0×) | 61.7 (0.7×) | 86.8 (1.0×) |
| 16 | 118.3 (1.0×) | 265.7 (2.3×) | **331.3 (2.8×)** |
| 24 | 96.8 (1.0×) | 407.5 (4.2×) | **515.6 (5.3×)** |

**说明**: [[HammerDB]] TPROC-C（即 [[TPC-C]] 风格）峰值吞吐 NewOrder transactions per minute（NOPM），受 **p95 NewOrder 延迟 ≤ 20 ms** 约束。倍数归一化到 Gen-1。配置：每 vCPU 16 个 warehouse，每个虚拟用户访问全部 warehouse，虚拟用户数在 1–512 的 2 的幂上扫描；聚焦整个工作集可放进 DRAM 的缓存场景。$^*$Gen-1 无 24 vCPU 规格，改用次近的 32 vCPU（同 384 warehouses 以保证可比）。

**关键发现**:
- **1 vCPU 时 Lakebase 吞吐是 Gen-2 的 2.6×**——因为 Gen-2 的存储扇出（storage fan-out）吃掉相当一部分核心，而 Lakebase 只做单路 quorum 复制写，把 CPU 留给事务处理。
- **24 vCPU 时 Lakebase 比 Gen-2 高 26%**，同时维持 p95 < 20 ms；4/16 vCPU 中间档领先 Gen-2 **22–43%**。
- 对 Gen-1 最高 **5.3×**；4 vCPU 档两者持平（该规模下都未打满存储路径），随核心数增加差距拉大——Gen-1 的额外核心争抢同一条存储 I/O 路径，且依赖 full-page-image WAL 记录保持久性，而 Lakebase 把页物化卸载到 S3 后的 PS，WAL 体积显著更小。
- 成本论证：AWS EC2 R8gd.4xlarge 按需约 **$1.18/hr**，而 10 GB S3 Standard 约 **$0.0003/hr**（差三个数量级），且 PUT/GET 请求费与 compaction 分层费即使在重摄入下也相对计算很小——因此**按 vCPU 吞吐作为成本效率的代理指标**。

### Table 2: Heap-write time speedup over COPY（Direct-to-Storage 相对 COPY 的堆写加速）

| Size | 5 GB | 10 GB | 50 GB | 100 GB | 500 GB | 1 TB |
|---|---|---|---|---|---|---|
| DTS | 2.6× | 3.4× | 6.2× | 7.9× | 31.3× | **73.3×** |

**说明**: 常规 `COPY` 路径把摄入串行化到单个 PostgreSQL writer 上，[[Direct-to-Storage]] 消除该瓶颈——Spark 执行器并行地把 PostgreSQL 兼容堆页直接写到对象存储。测试对象为含混合文本与整数列的 1-KB 行表。加速比随规模单调增长，**1 TB 时达 73.3×**，体现水平扩展特性。

---

## 实验

### 数据集与基准

| 基准/场景 | 规模 | 特点 | 用途 |
|---|---|---|---|
| [[HammerDB]] TPROC-C | 1/4/16/24 vCPU，每 vCPU 16 warehouse | 缓存场景（工作集全放 DRAM），p95 ≤ 20 ms 约束 | OLTP 吞吐对比（Table 1） |
| [[TPC-H]] | scale factor 10 | 22 条分析查询，同硬件对比执行引擎 | [[Direct-to-Storage|Direct Access]] 分析性能（Figure 11） |
| 生产股票交易负载回放 | min 1 / max 7 CPU，32 → 10K QPS | 日周期（开盘骤增） | 自动伸缩（Figure 10） |
| 生产遥测 | 跨度 700+ 小时 | 计算存活时长、分支深度、p95 启动时间 | 工作负载形态与启动性能（Figure 1/2/9） |
| DTS 堆写 | 5 GB – 1 TB | 1-KB 行、混合文本/整数列 | 批量摄入加速（Table 2） |

### 实现细节

- **计算虚拟化**：不用 Kubernetes 原生容器——理由有二：弱者主机–容器隔离是安全漏洞的主要来源；云托管 K8s 里新资源供给某些 VM 要几十到几百秒。改用 **[[NeonVM]]**（在 K8s 中跑 VM 的 API，支持**原地垂直扩容**与**热迁移**），每个计算是独立的 QEMU/KVM VM。VM 内加了**可伸缩的自定义缓存**，补足 PostgreSQL 静态 shared buffers 的不足。
- **缓存落在盘上**：primary 计算缓存用**操作系统 page cache** 而非普通内存分配——即使缓存能放进内存也这么做，因为在内存伸缩时普通分配会给内核分配器造成压力、更脆弱。盘支撑的缓存在伸缩时**同步**调整大小。
- **伸缩机制**（[[NeonVM]]）：CPU 伸缩通过 guest agent 启停单个 CPU；内存伸缩用 **virtio-mem** 平衡 movable/kernel 内存比例，并限制伸缩范围以避免碎片导致的降配失败；磁盘伸缩结合**协作式回收**（文件系统发 discard → 主机后备文件 hole punch）与对 swap/临时存储的**超分**。
- **伸缩策略**：autoscaler-agent 每 **5 秒**从每个计算拉一次指标（只保留最新），算法基本无状态（抑制抖动的状态放在计算侧，以更长窗口的负载度量暴露）。**线性均匀伸缩**，单位为 1 GiB RAM + 成比例 CPU 份额；CPU/内存/缓存的目标单元独立估算后**取最大值**作为 VM 尺寸。
  - **CPU 策略**：混合 guest 内核 1 分钟与 5 分钟负载均值——两者差异显著时用 1 分钟（求响应性），否则用 5 分钟（降抖动）。
  - **缓存策略**：自定义缓存维护**改造版 [[HyperLogLog]]**（bucket 的每个 bit 替换为"该 bit 有非零哈希值时的最新时间戳"），廉价近似"自过去某时刻起访问过的唯一页数"；用**平台期检测**算法在 1–60 分钟窗口上估计工作集；为避免二元阈值的抖动，最终缩放值是对所有切分点的**加权平均**（增长率外推到 1 小时，大工作集下给更早的点更低权重）。
  - **内存策略**：见公式 6。
  - **内存压力快路径**：guest agent 每 **100 ms** 轮询内存用量，一旦受压就主动向 autoscaler-agent 请求加内存，不理会最近的周期指标。
- **Scale to Zero**：周期性任务检查"距上次查询的时间"，超过用户配置超时（**默认 5 分钟**）即关闭计算；计算重启不保留任何状态，除了已持久化的内容；VM 的完全终止与清理异步进行。唤醒时 Proxy 先挂住入站 PostgreSQL 连接，再从池中分配计算（下载 basebackup 并启动），然后才开始转发连接，最后异步把 VM 调整到用户配置的最小规格。
- **Warm Pool**：池中 VM 规格**统一**，靠原地垂直扩容在分配时再调整，从而不必为不同用户的伸缩范围维护多个池；计算内二进制**预预热**以避免惰性内存分配或磁盘 IO 造成的启动延迟。生产观测到启动请求有季节性与突发尖峰，**用 ML 模型**预测需求来定池子大小。

### 与 Lakehouse 的集成

1. **统一治理（Unity Catalog）**：把每个调用者的 Databricks 有效身份经 OAuth 传播到 Lakebase，首次使用时映射为 PostgreSQL 角色，使 Spark 管道 / Model Serving 以自己的身份连接而无需单独数据库凭证。改造 [[Unity Catalog]] 把访问控制与执行下放给 Lakebase，管理员发标准 `GRANT SELECT` 会被翻译成原生 PostgreSQL 查询并**同步**应用；UC 的目录元数据也**实时**来自 PostgreSQL 系统表，因而 UC 视图始终反映数据库当前状态，消除治理层与运行库之间的 **ACL 漂移**。
2. **湖数据服务化（Synced Tables）**：标准路径用 Spark 流式 DataFrame 读 Delta change feed 更新 PostgreSQL 目标表；**[[Direct-to-Storage]]** 路径处理大规模快照装载——Spark 执行器构造 PostgreSQL 兼容页文件直写对象存储，只有元数据记录走 Lakebase 计算。原子性在**关系粒度**：Spark 先物化完整表页并上传，随后计算在一个事务里做**仅元数据的关系交换**；隔离性靠生成带"冻结可见性"元数据的页文件，使它们对被交换快照可见的事务立即可见；持久性靠提交前所有页已落 S3 + 关系交换 WAL 记录走标准 SK 复制。一致性上，**对正在做 DTS 快照装载的表并发 DML 会被拒绝**。未来可推广到分布式索引构建与物理 schema 变更。
3. **运行数据的分析访问**：数据管道方向用逻辑解码扩展 **wal2delta** 挂到 PostgreSQL 逻辑复制流，把 WAL 记录解码后以 **SCD Type 2** 格式直接追加到 Delta 表，下游 Spark 消费历史流或压实成快照；实时分析方向用 **Direct Access** 让 Spark / Lakehouse//RT 直接读共享存储上的页文件——Lakehouse//RT 在目标 LSN 取一致页，用 **[[SIMD]] 解析** + 简化 [[MVCC]] 检查，解析出的 Arrow 批次按 relation + page range 做内存缓存；数据变化时 PS 提供 **modified-blocks bitmap**（两个 LSN 之间变化页的 [[Roaring Bitmap]]），只刷新过期页。

### 可复现性评估

- [x] 代码开源（架构源自开源 [[Neon]]，DOI 可查）
- [ ] 预训练模型（不适用）
- [x] 实验配置描述较完整（TPROC-C 的 warehouse/用户扫描范围、p95 约束、TPC-H 的 SF 与同硬件对比均有交代）
- [ ] 数据集可获取（生产遥测数据不可获取）
- [ ] 对比系统匿名化：Gen-1/Gen-2 只以"第一代/第二代原型"出现，未点名具体产品（如具体云厂商的哪个规格），削弱了可复现性

---

## 批判性思考

### 优点

1. **架构层级的重新划分清晰且可证伪**。把 Gen-1/Gen-2/Gen-3 的判据落在一个可检验的维度上——**存储格式是否开放 + 数据是否只能被单一引擎访问**——而不是"是否做过读写分离"。这让"第三代"的说法有实际含义，也解释了为什么 Aurora 类系统仍属于第二代。
2. **存储格式的取舍诚实**。作者明确写"我们**不**声称该存储格式是 state-of-the-art，而是为**可预测性**调的"，并给出 $O(1)$ 写放大、$O(I)$ 读、$O(1+R+wP/N)$ 空间放大的显式刻画与生产实测（7×）。用**常数层数**换取放大行为的可预测性，是针对"每天百万级数据库、每库都可能有 PITR 历史"这种场景的合理工程判断。
3. **"计算启动不阻塞在 PS 追平 SK 上"是个关键设计**。primary 以 PS 的 last record LSN 作为 redo 点，不等 PS 追到 SK 最新——把一个原本串行的等待从关键路径上摘掉，这是亚秒启动的主要来源之一（配合 warm pool）。
4. **LwLSN 双 LSN 请求语义**解决了"按 current LSN 取页必须等 PS 追平"的固有延迟，是个干净的小设计：request LSN 定版本，not-modified-since LSN 定 PS 需要摄入的最低水位。
5. **Direct Access 承认失败案例**（Q8/Q19 因缺索引嵌套循环连接而变慢），并给出并发热点下的量化退化（1.76×）与"不影响 primary 写延迟"的边界说明——比只报喜的工业论文可信。

### 局限性

1. **OLTP 实验的覆盖面窄**。只测了**全工作集进 DRAM 的缓存场景**，这是对"存储分离"架构最有利的场景——因为此时 PS 与对象存储的延迟几乎被彻底掩盖（正文自己提到 LFC 命中率 > 98.5%）。工作集大于内存（存储分离架构的真正压力点）、以及写密集/长事务场景完全没有数据。
2. **对比对象匿名化**。"Gen-1/Gen-2"只说"有代表性的第一代/第二代原型"，读者无法复现也无法判断所选配置是否具有代表性；Gen-1 的 24 vCPU 档还退化成 32 vCPU，进一步削弱可比性。
3. **分支的代价被轻描淡写**。分支本身是 $O(1)$，但正文承认"**深分支层级会损害读性能，因为一次读要跨更多 timeline**"，而 Figure 2 显示长尾存在 64+ 乃至 500+ 深度的真实层级。论文没有给出跨 timeline 读放大的量化，也没有评估 detach（异步拷贝祖先数据）在大库上的实际耗时与对 PS 的负载影响。**且不支持分支合并**——这在"每次 prompt 都分支、事后挑选胜者"的 agent 场景里是个真实缺口。
4. **空间放大 7× 是生产实测但缺少分布与归因**。论文说 $wP/N$（PITR 保留的历史）通常主导，却没有给出 P/W/N 的实际分布、也没说明按 PITR 计费后用户实际为此付出的成本比例。
5. **多引擎写入开放的边界有限**。Direct-to-Storage 装载期间**拒绝并发 DML**，即"开放存储"在批量写入时仍需要一把粗粒度的锁；而且这条路目前只覆盖关系粒度的快照装载与（展望中的）索引构建/schema 变更。
6. **未讨论 PS 侧的 range-scan 缺失后果**。PS 只有 `GetPage@LSN` 点查 API，没有 range-scan 原语——所有扫描型负载都被推给计算侧缓存或外部分析引擎，意味着"非缓存型"访问模式（大表顺序扫描、备份、逻辑解码全量回放）的成本结构论文没有刻画。

### 潜在改进方向

1. **补充分支树的读放大模型与优化**：既然承认深分支损害读性能，可以引入"跨 timeline 的层索引合并"或对高频访问的祖先层做跨 timeline 缓存亲和；同时把 detach 做成可增量、可抢占的后台作业并给出对 PS 的 QoS 隔离。
2. **在 PS 上加入受限的 range-scan 原语**（例如按 key range + LSN 顺序吐出页并顺带做页级过滤），可显著降低全表扫描与备份/CDC 全量回放的跨层放大，也可缓解 Direct Access 在并发下的 1.76× 退化。
3. **分支合并的可行性**：即便物理合并困难，也可提供"逻辑上以某一分支为准、把另一分支的未冲突变更重放过去"的受控语义（例如按 relation 或主键区间判定冲突），让 agent 的多候选并行探索能收敛回来。
4. **工作集大于内存的场景评测**：这是存储分离架构的真实分水岭，应给出 LFC 命中率随工作集/内存比的曲线，以及 PS 与对象存储延迟在 p95/p99 上的可见度。
5. **公开 Gen-1/Gen-2 的具体配置**（至少给出规格档位、存储类型、复制方式），让 OLTP 结论可被第三方复核。

---

## 关联笔记

### 基于

- [[Neon]]: Lakebase 的计算存储分离架构、[[Safekeeper]]/[[Pageserver]] 拆分与 [[NeonVM]] 直接源自开源 Neon；论文致谢明确感谢 Neon 团队与社区。
- [[LSM-tree]]: PS 的层文件组织借用 LSM 的"内存缓冲 → 不可变文件 → 后台压实"骨架，但索引维度改为 **key × LSN 二维**且层数固定。

### 对比

- [[Amazon Aurora]]: Gen-2 代表。同样做计算存储分离，但存储格式私有、数据只能通过主引擎访问；Lakebase 在 24 vCPU 档吞吐为其 5.3×。
- Gen-1（多 AZ 同步复制盘上的 PostgreSQL）: 用于在**同等持久性语义**下做公平对比，Lakebase 最高 5.3×。
- Snowflake Unistore / Microsoft Fabric: 同样想把 OLTP 拉进湖仓体系，但 OLTP 引擎存储格式仍私有。

### 方法相关

- [[Database Branching]]: $O(1)$ 分支/克隆，是"数据库版 git checkout -b"的实现基础。
- [[PITR]] 与 [[Timeline]]: 分支、时间旅行查询与 GC 边界的统一抽象。
- [[LSN]]: 贯穿 Compute/SK/PS 三层的一致性坐标。
- [[GetPage@LSN]]、[[LwLSN Cache]]、[[Local File Cache]]: 读路径三件套。
- [[Safekeeper]]、[[Raft]]、[[Split-Brain]]: 写路径的共识与防脑裂机制。
- [[Direct-to-Storage]]、[[wal2delta]]、[[Unity Catalog]]、[[Lakehouse]]: 与湖仓集成的三条通路。
- [[Scale-to-Zero]]、[[Warm Pool]]、[[HyperLogLog]]: 弹性与亚秒启动的支撑。

### 硬件/数据相关

- [[Object Storage]]: 11 个 9 持久性的底层依赖，也是"开放格式"得以成立的前提。
- [[HammerDB]] / [[TPC-H]]: 论文使用的两个基准。
- [[Availability Zone]]: 高可用拓扑的三 AZ 单元。

---

## 速查卡片

> [!summary] Lakebase: Serverless Postgres over Open Lake Storage
> - **核心**: 把 PostgreSQL 全部数据（含 WAL）以开放页格式放到云对象存储，计算完全无状态化，得到 $O(1)$ 分支 + 亚秒启动 + scale-to-zero + 多引擎零拷贝访问。
> - **方法**: Safekeeper（类 Raft quorum 复制 WAL、term 防脑裂）+ Pageserver（key × LSN 二维 LSM 层文件、只暴露 GetPage@LSN/GetBaseBackup）+ NeonVM 原地垂直伸缩 + warm pool。
> - **结果**: 24 vCPU 下 TPROC-C 515.6 K NOPM（Gen-2 的 1.26×、Gen-1 的 5.3×）；p95 启动 < 500 ms；DTS 堆写 1 TB 时比 COPY 快 73.3×；TPC-H Direct Access 总时长降 10×；生产约 7× 空间放大，每日创建 1000 万+ 数据库。
> - **代码**: https://github.com/neondatabase/neon

---

*笔记创建时间: 2026-09-27 22:30*
