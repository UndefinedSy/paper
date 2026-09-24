---
title: "Highly Scalable SQL Azure MVCC Version Cleaner"
method_name: "AzureMVCCVersionCleaner"
authors: [Jithendra Gandikota, Prashanth Purnananda, Chaitanya Sreenivas Ravella, Adrian-Leonard Radu, Hanuma Kodavalla, Max Garton, Allison Ohara, Wayne Chen, Raghavendra TK]
year: 2026
venue: PVLDB 19(12)
tags: [mvcc, version-cleanup, garbage-collection, sql-azure, transaction-processing, cloud-database]
zotero_collection: VLDB/2026
image_source: local
created: 2026-09-24
---

# 论文笔记：Highly Scalable SQL Azure MVCC Version Cleaner

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | Microsoft |
| 日期 | 2026，PVLDB 19(12): 3916–3927 |
| 项目主页 | — |
| 对比基线 | [[Autovacuum]]（PostgreSQL）、[[InnoDB Purge]]（MySQL）、[[Oracle SMON]]（Oracle） |
| 链接 | [DOI](https://doi.org/10.14778/3827998.3828005) |
| 原始 PDF | `VLDB/2026/Azure MVCC Version Cleaner.pdf` |

---

## 一句话总结

> SQL Azure 通过读预取、动态多线程并行、版本链短路指针、Side List 重试、用户协助清理与事务级/语句级水位追踪，把 MVCC 版本清理从单线程全库扫描升级为可扩展、抗热点冲突、水位推进更快的后台系统，在千万级生产数据库上把满 sweep 时长从数天压缩到数小时。

---

## 核心贡献

1. **可扩展性改造**：引入 [[Read-Ahead]] 预取、动态[[Parallel Version Cleanup|并行清理]]和版本链短路指针，解决单线程清理在 [[Elastic Pool|Elastic Pool]] 与 Hyperscale（>100TB）超大库上的吞吐瓶颈。
2. **热点冲突下的鲁棒清理**：以 [[Side List]] 记录抢锁失败页并在 sweep 末尾重试，并引入 [[User-Assisted Cleanup]] 让持有冲突表锁的用户事务顺带完成清理，将热点工作负载下的 sweep 成功率提升到 98.6%。
3. **水位加速推进**：提出 [[Transaction-Level Version Tracking]] 与 [[Statement-Level Cleanup]]，把已提交事务产生的版本从「必须等最慢的中止事务/最老快照」的粗粒度水位中解耦出来，使可清理性判定精细到事务和语句级别。
4. **生产规模验证**：改动已在超过 1200 万个 Azure SQL 数据库上默认启用，论文用 30 天生产遥测量化了并行清理、事务级追踪、语句级清理各自贡献的 [[Persistent Version Store|PVS]] 回收量。

---

## 问题背景

### 要解决的问题

[[MVCC]] 系统在事务更新/删除行时会不断产生旧版本，这些旧版本若不及时清理会造成空间膨胀（space bloat）。SQL Azure 用 RCSI（Read Committed Snapshot Isolation，默认隔离级别）、[[Snapshot Isolation]]、[[Constant Time Recovery]]（常数时间恢复）和 Metadata Versioning 都依赖 MVCC 维护的多版本行，因此需要一个后台 **Version Cleaner（版本清理器）** 持续、及时地回收不再被任何这些特性需要的旧版本。论文明确把 Version Cleanup 与 Ghost Cleanup（只清理逻辑删除行）区分开，本文改动只针对 Version Cleanup。

### 现有方法的局限

此前的版本清理实现存在三类生产级痛点：

1. **单线程扫描慢**：面对 [[Elastic Pool|Elastic Pool]]（多库共享 CPU/IO）和 Hyperscale（单库可超 100TB）时，单线程清理满 sweep 有时耗时超过一周；同一行在单事务内被反复更新会形成很长的版本链，追踪到已提交版本的开销大，尤其在远程存储场景下容易触发 latch timeout。
2. **热点冲突下抢锁失败**：清理器被设计为低优先级、非侵入式，遇到无法立即获取的表/页锁会跳过并推迟到下一轮，持续高并发写入下会形成清理积压。
3. **水位推进被拖慢**：清理依赖 [[MinUsefulXTS]]（全局最小可用时间戳）、OldestActiveXdesId、OldestAbortedXdesId 等水位。长事务、空闲事务或中止事务的 undo 延迟都会拖慢水位推进，使本该可以清理的已提交版本被迫和最慢的事务绑定在一起继续占用空间。

### 本文的动机

论文强调这些改进不是零散补丁，而是围绕三条设计原则展开：清理必须随负载强度自适应可扩展、必须与前台事务协作而非阻塞、必须足够细粒度以避免长尾延迟。所有优化均来自对 Azure SQL 生产遥测中版本存储最大消费者的观察，聚焦于真实客户负载中对 PVS 增长的主要贡献者，而非追求 MVCC 理论上的完备清理。

---

## 方法详解

### SQL Azure 中的 MVCC 基础

SQL Azure 采用行版本化 + 时间戳的 [[MVCC]] 实现：

- **XTS（Transaction Timestamp）**：服务器级单调递增计数器，重启归 1；每个快照记录 read XTS 与创建时刻所有活跃事务的最小 XTS（MinXTS）。
- **XdesID（Transaction Id）**：写事务首次生成日志时分配的单调递增事务号。
- **CSN（Commit Sequence Number）**：写事务提交时被赋予的 commit XTS。
- **XdesHistory**：已提交事务 XdesID → CSN 的映射表。
- **[[Aborted Transaction Map|ATM]]**：记录所有已中止事务 XdesID 的结构。
- **[[MinUsefulXTS]] / MinUsefulStatementXTS**：分别是系统内所有快照 MinXTS 的最小值、所有 RCSI/SI 活跃语句 read XTS 的最小值，用作版本清理的可清理性阈值；二者更新有节流间隔。

插入时行只需打上 XdesID；更新会创建新版本并给旧行加 8 字节指针（指向旧版本的 page/slot id），同时打上 6 字节 XdesID；删除是逻辑删除，旧版本推入版本存储并留下 ghost stub。读操作沿版本链找到第一个 CSN < read XTS 的提交版本。

### [[Persistent Version Store]]（PVS）

旧版本存放在数据库内的 **持久版本存储（PVS）**：

- **[[In-Row Versioning|in-row 版本]]**：旧版本作为 diff 嵌入当前版本同一页内，省去额外页读，但会增加行大小、可能诱发 B-Tree 页分裂，因此 diff 与行大小都设了上限。
- **[[Off-Row Versioning|off-row 版本]]**：超出 in-row 限制的版本存入一张全库共享、面向 append-only 优化的内部无索引表，按 (Page ID, Slot ID) 物理定位查找；生成版本即插入该表的一行，回收版本即回收整页。

图1（Figure 1）展示了行版本链的存储结构。

### 原有清理流程（Baseline）

后台清理任务周期性（约几分钟一次）逐库执行：

1. 复制当前 ATM 快照；
2. 通过 [[Page Free Space (PFS)|PFS]] 元数据页定位含版本的数据页；
3. 对被中止事务修改过的行做 [[Logical Revert|逻辑回滚]]；
4. 从 ATM 移除已完成回滚的事务；
5. 清理不再需要的 off-row PVS 版本；
6. 处理下一个数据库。

**[[Page Free Space (PFS)|PFS]] 扫描**：每个 PFS 页跟踪 8088 个数据页的 1 字节版本标记，数据页本地也缓存同一状态位，避免每次都访问 PFS。

**[[Logical Revert]]**：清理器对页/表加意向排他锁，依据行上 XdesID 比对 ATM 判断是否需要回滚；同时清理不再需要的 in-row 版本；若某页所有版本已清空则清除其 PFS 版本位。为应对页分裂导致的行迁移，系统维护 `MinXdesIdInMove`（本轮迁移行中最小 XdesId），sweep 结束时只从复制的 ATM 里移除 XdesId 低于该值的中止事务，以保守方式保证正确性。

**off-row 清理**：借助 Accelerated Database Recovery 维护的内存有序表 **PVSPageTrackerList** 按 Page Id 跟踪每个 PVS 页的最高事务号 `PageMaxXdesId`（利用 XdesID 单调递增只存最大值节省内存）。当页写满且 `PageMaxXdesId` 低于 `OldestActiveXdesId`、`OldestAbortedXdesId`，且其 CSN 低于 [[MinUsefulXTS]] 时，该页可被回收。故障恢复时该内存表通过重新扫描 PVS 表重建，保守地把所有页归入「中止」列表。

### 挑战一：可扩展性（Section 3.1）

单线程清理在 [[Elastic Pool]]、Hyperscale（单库可超 100TB）以及长版本链场景下均成为瓶颈，且 sweep 一旦滞后，下一轮待清理量会滚雪球式增长。三项针对性优化：

#### [[Read-Ahead]]（读预取）

清理器先批量收集所有版本位被置位的页，再以轮询方式跨文件发起预取 I/O，减少逐页读的磁盘等待延迟，且避免单一文件被预取请求打满。

#### [[Parallel Version Cleanup|动态并行清理]]

引入自适应并行：仅当资源充足且 PVS 体量超过阈值时才在库内并行化 sweep；体量降回阈值以下则恢复单线程。跨库并行与库内并行都遵循此策略，并设 CPU 使用上限，避免抢占用户负载资源。生产观察满 sweep 时长从数天降到数小时。

#### 版本链短路指针（Figure 2）

针对同一行在单事务内被反复更新、事务最终中止的场景：off-row 版本被推入版本存储时，除了指向链中「前一版本」的指针，还额外存一个指向该行「第一个已提交版本」的指针，复用已有但冗余的元数据字段编码，不增加每版本存储成本。这样从最新版本 $v_{lat}$ 出发，只需读 $v_{lat} \to v_{100}$（off-row 第一个版本）再一跳到 $v_1$（首个提交版本），遍历长度恒为 3，而不是原本长度为 100 的完整链。

### 挑战二：热点写工作负载导致的锁冲突（Section 3.2）

清理器以低优先级运行、不与用户事务抢锁，遇到无法立即获取的锁会跳过并推迟到下一轮；持续高并发下会形成滞留版本积压。此外页分裂可能把中止行搬到清理器已扫过的页范围，如果不能在单轮 sweep 内清理完所有已复制 ATM 关联的版本，ATM 中中止事务移除会被延迟。

#### [[Side List]]

所有因抢锁失败被跳过的页，以及因页分裂而被移入的、含中止行的页，都记录到专门的 side list 中。整库 sweep 完成后，主清理线程在配置时限内循环重试 side list 中的页。为防极端持续冲突下 side list 无限增长，该列表设有上限；超限时本轮 sweep 会被判定失效并推迟到下一轮（论文称此情况罕见）。

#### [[User-Assisted Cleanup]]

即便有 Side List 重试，仍存在少数场景因拿不到表级锁而持续清理失败，通常发生在某工作负载长时间持有冲突锁的情况下。解决方案是把清理工作**委托给持有该冲突锁的用户事务**：清理器维护一张哈希表，key 为锁资源 ID，value 是该资源上被跳过的页列表；用户事务每条语句结束时检查自己持有的锁资源 ID 是否在该表中，若在则清理其中一部分页（数量有上限，防止拖慢前台语句）。该哈希表本身也设容量上限，超限的页转入 Side List 在 sweep 末尾重试。事务内清理操作在中止场景下是幂等安全的，失败会通过 Side List 机制被重试，不引入正确性问题。

Figure 3 展示了该机制的一个配套优化：当用户事务更新一行且该行当前版本是中止版本时，不必先走按需逻辑回滚，而是**直接用新版本覆盖中止版本**，同时把新版本链接到最后一个已提交版本，使这次更新的开销接近「该行从未存在中止版本」的正常路径。

### 挑战三：清理水位推进缓慢（Section 3.3）

清理水位由 `OldestActiveXdesId`、`OldestAbortedXdesId` 和 [[MinUsefulXTS]] 共同决定。长运行/空闲事务会拖慢 `MinUsefulXTS` 推进；sweep 滞后又会拖慢 ATM 中中止事务移除，进而拖慢 `OldestAbortedXdesId` 推进。二者叠加会强迫清理器保留本可安全删除的版本，PVS 持续膨胀。

#### 场景示例（Figure 4）

两个事务 T1（XdesId=100）、T2（XdesId=200）分别在 PVS 页 P1、P2 上生成 off-row 版本，各页 `PageMaxXdesId` 更新为 100、200。T1 在 XTS=300 中止并加入 ATM，`OldestAbortedXdesId` 更新为 100。T2 在 XTS=350 提交，此后系统无其他活跃事务，`MinUsefulXTS` 会推进到 350 之后。

**旧方案的清理行为**：因为水位卡在 `OldestAbortedXdesId=100`，P1、P2 都无法清理，直到 T1 逻辑回滚完成、`OldestAbortedXdesId` 推进为止——即便 P2 上的版本全部来自已提交事务、根本不需要用于回滚。中止事务的存在或其推进延迟会连带阻塞不相关的已提交版本清理。

#### [[Transaction-Level Version Tracking]]

引入事务级 PVS 页跟踪：每个事务维护自己插入版本所使用的 PVS 页列表，生成新版本时优先复用该列表中的页，无合适页才分配新页。

- **提交时**：该事务写满的 PVS 页转入面向已提交事务的全局 `PVSPageTrackerList`，末尾未写满的页可被其他事务复用；未写满页也在提交/回滚时被缓存以便复用。
- **中止时**：写满的页转入面向中止事务的全局列表；最后一页若混有中止版本，其 tracker 条目会置位 `HasAbortedTranVersions` 标志。

清理判据据此分流：

- **中止列表中的页**：仍按旧逻辑，用 `OldestAbortedXdesId`、`OldestActiveXdesId` 作水位。
- **已提交列表中的页**：若 `HasAbortedTranVersions` 置位，仍需两个水位（因为页上混有需要逻辑回滚的中止版本）；若未置位，清理**不再依赖 `OldestAbortedXdesId`**，因为页上所有版本都无需用于逻辑回滚。

回到示例：P1（T1，中止列表，`HasAbortedTranVersions` 未置位，假设无空闲空间）清理仍需等待 T1 undo 完成、ATM 截断，用 `OldestAbortedXdesId=100` 作水位；P2（T2，已提交列表，`HasAbortedTranVersions` 未置位、无空闲空间）可直接清理，不必和 `OldestAbortedXdesId` 比较——此时 `OldestActiveXdesId` 和 [[MinUsefulXTS]] 因无其他活跃事务而远大于所需值。

该机制的代价是适中的：原本全局维护的 PVS 页列表现在按事务维护，再转入两个全局列表，带来轻量开销。若崩溃导致事务级列表丢失，恢复时通过扫描 PVS 表重建全部 PVS 页集合，并保守地把它们全部放入中止事务列表，随后后台清理器照常按两个水位恢复常规清理。清理仍需比较页最高事务号的 CSN 与 [[MinUsefulXTS]]，确保活跃扫描仍在使用的页不会被清理。

#### [[Statement-Level Cleanup]]

判断 PVS 页版本是否仍被读事务需要，此前是用页上最高 XdesId 对应的 CSN 与 [[MinUsefulXTS]] 比较。但在 RCSI（默认隔离级别）下，可见性快照是**逐语句**建立的，只需保留「本语句开始时刻」需要的版本，而不必保留「整个 RCSI 事务开始时刻」需要的版本——后者正是 [[MinUsefulXTS]] 所代表的更保守边界。

该优化只适用于「页上所有版本都来自同一个事务」的场景（因为只跟踪最高 XdesId，无法在多事务混合页上确定所有事务的提交顺序），而这恰好是事务级 PVS 之后更常见的情形：`PVSPageTrackerList` 中用一个 bit 标记某页上的版本是否都属于同一（最高 XdesId 对应的）事务。对这类页，清理先用 CSN 与 [[MinUsefulXTS]] 比较，失败后再用更激进的 `MinUsefulStatementXTS` 重试。该功能与事务级 PVS 结合后显著提升了清理效率。

---

## 关键公式

论文以工程系统设计与生产遥测为主，**未给出编号数学公式**；核心量化关系体现为水位比较条件，摘录如下便于查阅：

**off-row PVS 页可回收条件**：

$$
\text{PageMaxXdesId} < \text{OldestActiveXdesId} \ \wedge\ \text{PageMaxXdesId} < \text{OldestAbortedXdesId} \ \wedge\ \text{CSN}(\text{PageMaxXdesId}) < \text{MinUsefulXTS}
$$

**含义**：只有当一页上最新写入者的事务号同时低于当前所有活跃事务、所有中止事务的最老事务号，且其提交时间戳低于全局最小可用时间戳时，该页才不再被 MVCC 可见性或逻辑回滚需要，可安全整页回收。

**符号说明**：
- $\text{PageMaxXdesId}$：某 PVS 页上插入版本的最高事务号；
- $\text{OldestActiveXdesId}$：所有活跃事务中最小的事务号；
- $\text{OldestAbortedXdesId}$：所有已中止事务中最小的事务号；
- $\text{MinUsefulXTS}$：所有快照中最小的 MinXTS，代表全局最小可用时间戳。

**[[Transaction-Level Version Tracking]] 下已提交页的放松条件**（`HasAbortedTranVersions` 未置位时）：

$$
\text{CSN}(\text{PageMaxXdesId}) < \text{MinUsefulXTS}
$$

**含义**：一旦页上所有版本都已知来自提交事务（无需逻辑回滚），清理判据可以去掉 `OldestAbortedXdesId` 这一约束，不再被慢速中止事务水位拖慢。

**[[Statement-Level Cleanup]] 的放松阈值**（单事务页失败重试）：

$$
\text{CSN}(\text{PageMaxXdesId}) < \text{MinUsefulStatementXTS}
$$

**含义**：RCSI 下用逐语句最小可用时间戳替代逐事务的 [[MinUsefulXTS]]，把可清理边界从"整个事务开始"收紧到"每条语句开始"，使已提交版本更早失去可见性需求。

---

## 关键图表

<!-- 图片：下载到本地 assets/ 文件夹，用 ![[]] wikilink 嵌入 -->

### Figure 1: MVCC 行版本链示例

![[assets/AzureMVCC/AzureMVCC_fig1_version_chain.png]]

**说明**：数据页 Page 15 上 Slot 5 的行携带 Tran ID 与指向 [[Persistent Version Store|Version Store]] 的 Version Locator；版本存储中每条版本记录同样含 Row Version、Tran ID 与指向下一（更旧）版本的 Locator，链尾以 NULL 结束，构成经典的单向版本链。

### Figure 2: 版本链短路指针优化

![[assets/AzureMVCC/AzureMVCC_fig2_version_chain_linkage.png]]

**说明**：长度为 100 的版本链中，每个 off-row 版本除保留到「前一版本」的常规指针外，额外携带一条指向「首个已提交版本 $v_1$」的短路指针。[[Logical Revert]] 时从最新版本 $v_{lat}$ 出发，经 $v_{100}$ 一跳直达 $v_1$，遍历深度恒为 3 而非 100。

### Figure 3: 覆盖中止版本的优化

![[assets/AzureMVCC/AzureMVCC_fig3_overwrite_aborted_version.png]]

**说明**：Page 15 上原本是 Committed Version + Aborted Version；新事务更新该行时不走按需逻辑回滚，而是直接用 New Version 覆盖 Aborted Version 的位置，并把 New Version 链接到 Committed Version，使该更新路径的开销接近无中止版本时的正常更新。

### Figure 4: 事务级追踪场景的内部状态

![[assets/AzureMVCC/AzureMVCC_fig4_internal_state.png]]

**说明**：ATM 中记录中止事务 XdesId=100；XdesHistory 记录 (XdesId=100, CSN=300) 与 (XdesId=200, CSN=350)；PVS 页 P1 由 T1（XdesId=100）写入，页 P2 由 T2（XdesId=200）写入。该状态是 [[Transaction-Level Version Tracking]] 章节示例的初始快照，用于对比旧/新两种清理判据下 P1、P2 能否被回收。

### Figure 5: 并行清理 vs. 非并行清理的 sweep 时长对比

![[assets/AzureMVCC/AzureMVCC_fig5_sweep_time_parallel.png]]

**说明**：在 PVS ≥100GB 的数据库上，[[Parallel Version Cleanup|并行清理]]将 P90/P99/P99.99 sweep 时长分别从 140s/5,641s/515,121s 降到 96s/1,335s/79,245s；尾部（P99.99，约 6 天 → 约 1 天）改善最显著，说明并行化对长尾大库的收益远超均值场景。

### Figure 6: 事务级清理与总清理量对比（30 天生产遥测）

![[assets/AzureMVCC/AzureMVCC_fig6_txn_level_cleanup.png]]

**说明**：每个数据点是 sweep 时长 ≥4 小时场景下全区域 PVS 回收量（TB）的聚合。30 天窗口内，[[Transaction-Level Version Tracking]] 贡献了约 60% 的总 PVS 回收量，说明在慢 sweep 场景下事务级追踪是回收空间的主要来源。

### Figure 7: 语句级清理与总清理量对比（30 天生产遥测）

![[assets/AzureMVCC/AzureMVCC_fig7_stmt_level_cleanup.png]]

**说明**：每个数据点是因 [[MinUsefulXTS]] 造成 PVS 持有量 ≥50GB 场景下全区域 PVS 回收量（TB）的聚合。30 天窗口内，[[Statement-Level Cleanup]] 贡献了约 27% 的总 PVS 回收量。

### Table 1: 常用缩写词表（Appendix 8.1）

| Acronym | Full form |
|---|---|
| MVCC | Multi Version Concurrency Control |
| RCSI | Read Committed Snapshot Isolation |
| SI | Snapshot Isolation |
| PVS | Persistent Version Store |
| XTS | Transaction Timestamp |
| XdesID | Transaction ID |
| CSN | Commit Sequence Number |
| PFS | Page Free Space |
| ATM | Aborted Transaction Map |

**说明**：论文附录的术语表，覆盖全文所有核心缩写。

---

## 实验

论文以生产遥测为主要证据来源，而非合成 benchmark：

### 验证与评测方法

- **回归验证**：所有改动先在 TPCC、TPCE 上验证无性能回退。由于两者事务短、PVS 体量小、达不到清理阈值，**不适合**评估清理效果本身，仅用于确认无回归。
- **效果评估**：全部效果数据来自 Azure SQL 生产环境的真实遥测，覆盖跨区域、跨规模的实际客户负载，用于衡量在持续版本存储压力下各优化的贡献。

### 关键生产数据

| 指标 | 数值 |
|---|---|
| 已启用改进的数据库数量 | 超过 1200 万个 |
| 并行清理生效阈值 | PVS > 100 GB，且资源充足 |
| 无并行清理时 P99.99 sweep 时长 | 约 6 天（515,121s） |
| 有并行清理时 P99.99 sweep 时长 | 约 1 天（79,245s） |
| 事务级追踪贡献的 PVS 回收占比（慢 sweep 场景，30 天） | 约 60% |
| 语句级清理贡献的 PVS 回收占比（MinUsefulXTS 持有场景，30 天） | 约 27% |
| 10 天内总 cleanup sweep 数 | 约 3.85 亿次 |
| 受冲突负载影响的 sweep 数 | 约 376 万次 |
| 受影响 sweep 中最终成功的数量 | 约 323 万次（成功率 98.6%） |
| 最终失败的 sweep 数 | 约 5.2 万次（绝对数仍偏高） |

### 定性观察

- 更快的清理直接降低客户存储成本（COGS）与 PVS 页跟踪所需内存。
- 大事务插入大量数据后中止的场景下，更快清理中止版本减少了不必要的页读，带来适度的查询性能提升。
- 客户报告的相关生产事件数量明显下降，即便 Azure 数据库总量持续增长。

---

## 批判性思考

### 优点

1. **工程改动均以生产遥测为驱动**：三类挑战（可扩展性、热点冲突、水位推进）均来自对真实客户负载中 PVS 增长主要贡献者的观察，而非纯理论上的 MVCC GC 问题，改动的收益也用 30 天生产数据量化，可信度和落地价值都较高。
2. **细粒度水位下放解决"一慢全慢"问题**：[[Transaction-Level Version Tracking]] 和 [[Statement-Level Cleanup]] 的共同思路是把粗粒度、全局共享的水位（`OldestAbortedXdesId`、[[MinUsefulXTS]]）下放到事务/语句粒度，使已提交、已不被读需要的版本不再被最慢的中止事务或最老快照拖累，是一个可迁移到其他 MVCC 系统的设计范式。
3. **对写路径侵入的克制**：[[User-Assisted Cleanup]]、覆盖中止版本优化等改动都严格限定了每次操作的清理配额，保证前台事务延迟不被明显拉高，体现了"清理器必须与前台协作而非争抢"的设计原则在工程上被认真落实。

### 局限性

1. **User-Assisted Cleanup 仅覆盖排他锁场景**：论文自己承认该特性目前只在用户事务持有表级 X 锁时生效，持有共享锁的事务无法触发，作者估计这限制了对 5.2 万次仍失败 sweep 的进一步改善（Section 4.4、Section 5）。
2. **Statement-Level Cleanup 限于单事务页**：目前只能应用于「所有版本均来自单一事务」的 PVS 页，因为仅跟踪最高 XdesId 无法确定多事务混合页的提交顺序，论文将扩展到多事务页列为未来工作（需要额外维护 LatestXdesId）。
3. **缺乏与开源系统的量化对比实验**：Related Work 部分与 [[Autovacuum]]、[[InnoDB Purge]]、[[Oracle SMON]] 只做了架构层面的定性比较，并未提供在相同负载下的吞吐/延迟/空间回收速度的直接对照实验，读者难以判断改进后的 SQL Azure 相对这些系统的绝对优劣。
4. **Side List 与容量上限的权衡未量化**：论文提到 Side List 超限会导致整个 sweep 被判定失效并推迟，但没有给出该情况发生的频率或对系统的实际影响，"论文称此情况罕见"缺少数据支撑。

### 潜在改进方向

1. 将 [[User-Assisted Cleanup]] 扩展到共享锁场景，可能需要更细粒度的行级/页级协作机制而非表级锁资源 ID 索引。
2. 通过额外维护 LatestXdesId（而非只有 PageMaxXdesId）把 [[Statement-Level Cleanup]] 推广到多事务混合页，论文已指明方向但未给出具体设计。
3. 将 Ghost Cleanup 与 Version Cleanup 合并为统一后台清理器，避免两者各自扫描、加锁带来的冗余 I/O（论文列为未来工作）。
4. 当清理器长期拿不到锁时，可在共享 latch 下读页计算「中止事务涉及行的最小 XdesId」，实现 best-effort 清理而非让整轮 sweep 失效（论文列为未来工作）。

### 可复现性评估

- [ ] 代码开源
- [ ] 预训练模型（不适用）
- [x] 训练细节完整（作为系统论文，机制描述完整，但缺少可复现的开源实现或独立 benchmark 脚本）
- [ ] 数据集可获取（生产遥测数据未公开，仅以图表汇总形式披露）

---

## 关联笔记

### 基于

- [[Constant Time Recovery]]：本文改进的持久版本存储（PVS）正是从该工作把版本存储从 TempDB 迁移进用户数据库而来。
- Socrates: The New SQL Server in the Cloud（SIGMOD 2019）：SQL Azure 计算-存储分离架构的基础背景。

### 对比

- [[Autovacuum]]：PostgreSQL 的周期性后台清理，多进程并行但单表/索引不可并行，且需应对 32 位事务号 wraparound。
- [[InnoDB Purge]]：MySQL InnoDB 的持续增量 purge 线程，回滚同步发生在用户事务而非后台。
- [[Oracle SMON]]：Oracle 的 undo 空间清理，采用「保留期 + 按需覆写」策略而非主动删除。

### 方法相关

- [[MVCC]]：全文围绕的核心并发控制模型。
- [[Persistent Version Store]]：版本清理的直接对象。
- [[Transaction-Level Version Tracking]]：解决水位推进缓慢的核心机制之一。
- [[Statement-Level Cleanup]]：在 RCSI 下进一步收紧可清理边界的机制。
- [[Side List]] / [[User-Assisted Cleanup]]：应对热点写冲突的两级重试与协作机制。

### 硬件/数据相关

- 生产环境：Azure SQL 千万级数据库、Elastic Pool、Hyperscale（单库可超 100TB）。
- 回归基准：TPCC、TPCE（仅用于确认无性能回退，不用于评估清理效果）。

---

## 速查卡片

> [!summary] Highly Scalable SQL Azure MVCC Version Cleaner
> - **核心**：把 SQL Azure 的单线程 MVCC 版本清理器改造为可扩展、抗冲突、水位下放到事务/语句级别的后台系统。
> - **方法**：Read-Ahead + 动态并行清理 + 版本链短路指针（可扩展性）；Side List + User-Assisted Cleanup（热点冲突）；Transaction-Level Version Tracking + Statement-Level Cleanup（水位推进）。
> - **结果**：P99.99 sweep 时长从约 6 天降到约 1 天；事务级/语句级清理分别贡献约 60%/27% 的额外 PVS 回收；千万级数据库上生产落地，热点负载下 sweep 成功率 98.6%。
> - **代码**：未开源。

---

*笔记创建时间: 2026-09-24*
