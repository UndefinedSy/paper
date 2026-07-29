---
title: "ByteGraph-Dione: An Adaptive Dual-Format Graph Engine with Hotspot Awareness and Transaction Efficiency for Production-Scale Workloads"
method_name: "ByteGraph-Dione"
authors: [Chao Chen, Cheng Chen, Huiming Zhu, Shijiao Yang, Yingqian Hu, Wei Wang, Wei Zhang, Bingyu Zhou, Miaomiao Cheng, Lei Xiao, Ping Yang, Ye Wang, Shuai Zhang, Lei Zhang]
year: 2026
venue: SIGMOD-Companion
tags: [graph-database, htap, adaptive-storage, mvcc, hotspot-scaling, snapshot-isolation, production-system]
zotero_collection: SIGMOD/2026
image_source: local
created: 2026-07-29
---

# 论文笔记：ByteGraph-Dione

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | ByteDance Inc. |
| 日期 | May 31–June 5, 2026 |
| 项目主页 | - |
| 对比基线 | [[ByteGraph]]、DB-X、DB-Y |
| 链接 | [DOI](https://doi.org/10.1145/3788853.3803073) |

---

## 一句话总结

> ByteGraph-Dione 面向生产级动态图 HTAP，以长事务聚合、邻接表级行列自适应和热点数据选择性复制，在降低 SI、存储冗余与副本重建开销的同时显著提升混合负载吞吐。

---

## 核心贡献

1. **生产级图负载画像**：揭示同一图上 OLTP/OLAP 比例随时间和业务剧烈变化，热点遵循幂律、强度差异大且可持续半小时以上。
2. **[[Lightweight Transactions for Hybrid Workloads|轻量级混合事务]]**：聚合长 OLAP 事务所需版本，短 OLTP 版本仅在内存保留固定窗口；在提供 [[Snapshot Isolation|SI]] 时将吞吐损失从 35.5% 降到 16.49%。
3. **[[Adaptive Dual-Format Graph Storage|自适应双格式图存储]]**：同一 [[BW-Tree]] LeafPage 支持 RowBlock/ColumnsBlock，并在单条邻接表粒度依据在线成本模型无停机转换。
4. **[[Fine-Grained Hotspot Scaling|细粒度热点扩展]]**：只在只读副本上重建热点 key 的页面，并把请求路由到已缓存对应热页的副本，避免全量日志回放和缓存污染。
5. **生产级收益**：相对两个商业图数据库吞吐提升约 20–40×；相对上一代 ByteGraph，在不同负载下提升约 35–80%。

---

## 问题背景

### 要解决的问题

ByteDance 的 Douyin 等业务每天产生数十亿节点、数百亿边。同一张图既承载一跳 Top-K、增删边等低延迟 OLTP，也承载多跳遍历、路径查询等最长约 30 分钟的 OLAP。系统至少要提供 SI，同时在成本约束下适应负载比例和热点位置的持续变化。

### 现有方法的局限

- **传统 [[MVCC]]**：长 OLAP 持有旧快照，使短 OLTP 产生的历史版本也不能回收；高更新率下造成磁盘 I/O 放大和长版本链 CPU 开销。
- **静态存储格式**：行存适合 OLTP 完整记录点读，列存适合 OLAP 少列扫描。上一代 ByteGraph 固定行存，无法兼顾大范围分析。
- **双份行列副本**：性能好但空间接近翻倍；对大量低流量小邻接表也保存两份数据不经济。
- **粗粒度只读副本**：为少数热点 key 回放整个节点 WAL、重建全部页面；每个副本缓存全量数据，造成 CPU、内存浪费和低命中率。

### 负载洞察

1. **比例动态**：社交互动 OLTP 占 75%–95%，关系图占 65%–90%，数据血缘 OLAP 占 70%–90%；同一集群一天内还会继续变化。
2. **查询差异**：OLTP 以 1-hop、十亿级 QPS、亚秒延迟、RC 为主；OLAP 以 2+ hops/path query、十万级 QPS、最长 30 分钟、SI 为主。
3. **热点动态**：多个高连接节点可同时变热，第一名热点可比普通节点高数十倍，并持续至少半小时。
4. **格式差异**：典型 OLTP 中行存吞吐高 1.65×；典型 OLAP 中列存吞吐高 2.02×。

---

## 方法详解

### 系统架构

ByteGraph-Dione 延续 [[Disaggregated Storage-Compute Architecture|存算分离]]：Execution Layer 负责图查询解析与执行；Storage Engine Layer 在 append-only 共享云存储上运行 BW-Tree forest。每个 vertex 对应一棵 BW-Tree，出边按 key 排序组成 [[Adjacency List|邻接表]]。系统在此基础上加入三个模块：

1. Lightweight Transactions：分开管理长 OLAP 与短 OLTP 的版本生命周期。
2. Adaptive Dual-Format：在邻接表粒度选择 row/column 格式。
3. Fine-Grained Scaling：按热点 key 复制页面并定向路由流量。

### 模块 1：[[Lightweight Transactions for Hybrid Workloads]]

#### 长 OLAP 事务聚合

- 将可能长时间运行的查询（如超过两跳）识别为 OLAP，并按时间窗口批量执行。
- 同一窗口内多个更新不必各自为长事务保留持久版本，只持久化窗口末端需要的版本。
- OLAP 所需版本标为 persistent；内存紧张时可换出到共享存储。
- 更长窗口减少更多历史版本，却会增加等待时间并带来轻微结果时效误差；论文默认 30 秒。

#### 短 OLTP 内存 MVCC

- 短事务到达即执行，其独立历史版本只保存在内存，避免从存储取回旧版本。
- 版本默认保留 30 秒，足以覆盖两跳以内 OLTP；超时且无人持有即可回收。
- 该分治把“分析快照的持久历史”和“事务点读的短命历史”解耦，缩短版本链。

### 模块 2：[[Adaptive Dual-Format Graph Storage]]

#### 统一 LeafPage 抽象

- **Row Format**：属性连续存入 RowBlock；Pointer Array 指向每个 key 的最新版本，rollback pointer 串联历史版本。
- **Column Format**：类似 [[Apache Arrow]]，ColumnsBlock 按列组织属性；每次更新追加新 block，旧 block 只读，定期 compaction。逻辑指针由 block ID + offset 构成。
- 两种格式共享 Pointer Array、commit/delete timestamp、rollback pointer、MVCC、过滤式行扫描和列扫描接口。

#### 查询路径

- **OLTP 取 K3 全属性**：两种格式都先由 BW-Tree 找 LeafPage，再用 Pointer Array 找最新合法版本；行存属性连续，列存需跨列地址计算。
- **OLAP 按时间过滤**：行存逐行解析属性；列存只扫描过滤列、commit timestamp、delete timestamp，再访问命中行所需列。

#### 无停机在线转换

以 row → column 为例：

1. 新更新进入专用 append-only page。
2. 对原行存邻接表做实时快照。
3. 将快照转换为列存。
4. 回放第一阶段积累的增量更新。
5. 接近追平后进入双写，完成切换。

转换单位是单条邻接表，避免整个 shard 同时转换引起性能抖动。

#### 在线格式决策

系统从生产 trace 学习行/列格式下扫描行数、访问属性数、定位版本和固定 I/O 的平均成本。若列存处理大量索引式行扫描的估计成本显著更高，就转为行存；若频繁少列扫描使行存成本更高，就转为列存。只有成本差超过转换阈值才切换，避免来回抖动。

### 模块 3：[[Fine-Grained Hotspot Scaling]]

- 监控每条邻接表/每个 key 的访问频率，而不是仅监控节点总负载。
- 新建只读 follower 后，仅 lazy-apply 热点 key 的 WAL 并重建热页；冷 key 不在该副本构页。
- 路由层记录热点副本集合，把请求发送到已缓存目标热页的节点。
- 热度进一步倾斜时，可为最热 key 单独增加 follower。例如 key 6–8 为 200 QPS 时由 RO1 承载，key 8 升至 800 QPS 后新增 RO2 专门承载 key 8。
- 该方法同时减少 page reconstruction、缩小 working set，并提升 buffer pool 命中率。

---

## 关键公式

论文只有一个编号公式，由行存总成本与列存总成本组成。

### 公式 1：[[Row-Column Cost Model|邻接表行列格式成本模型]]

$$
\begin{aligned}
Cost_{row}
&=
\sum_{i=1}^{Q}
\left(
 a_{row}N_{rows,i}N_{attr,i}
 +BN_{rows,i}
 +C_{row}
\right),\\
Cost_{col}
&=
\sum_{i=1}^{Q_{olap}}
\left(
 a'_{col,olap}N_{rows,i}N_{attr,i}
 +BN_{rows,i}
 +C'_{col,olap}
\right)\\
&\quad+
\sum_{i=1}^{Q_{oltp}}
\left(
 a'_{col,oltp}N_{rows,i}N_{attr,i}
 +BN_{rows,i}
 +C'_{col,oltp}
\right).
\end{aligned}
$$

**含义**：在统计窗口内分别估计某邻接表使用行存和列存执行全部查询的代价，选择更低者；列存因 OLTP 跨列取完整行与 OLAP 少列扫描的单位代价不同而分项建模。

**符号说明**：
- $Q$：窗口内查询总数，$Q=Q_{oltp}+Q_{olap}$。
- $N_{rows,i}$：查询 $i$ 扫描的行数。
- $N_{attr,i}$：查询 $i$ 访问的属性数。
- $a_{row}$：行存处理一个“行×属性”的平均单位代价。
- $a'_{col,oltp}$、$a'_{col,olap}$：列存执行 OLTP/OLAP 时不同的单位代价。
- $B$：页内二分定位数据行并寻找满足 MVCC 的版本的平均代价。
- $C_{row}$、$C'_{col,*}$：统一吸收页定位、cache 和 I/O 等固定开销。

---

## 关键图表

### Figure 1：The Architecture of Previous ByteGraph / 上一代 ByteGraph 架构

![[assets/ByteGraph-Dione_fig1_previous_architecture.png]]

**说明**：执行层与存储层解耦；存储层用 BW-Tree forest、WAL 和 leader/follower，在 append-only 共享存储上提供图访问。

### Figure 2：The Throughput of OLTP and OLAP in Douyin Cluster / 日内混合负载

![[assets/ByteGraph-Dione_fig2_daily_workload.png]]

**说明**：OLTP 与 OLAP 的绝对吞吐和比例随时间变化，静态格式与固定资源配置无法始终最优。

### Figure 3：Comparison of Hotspot Changes / 热点强度变化

![[assets/ByteGraph-Dione_fig3_hotspot_changes.png]]

**说明**：Rank 1/10/100 与普通用户流量跨多个数量级，热点不仅偏斜而且持续存在。

### Figure 4：Transaction Challenges in Hybrid Workloads / 混合事务版本膨胀

![[assets/ByteGraph-Dione_fig4_mvcc_challenge.png]]

**说明**：长 OLAP 与短 OLTP 交错，使多个更新版本长期被不同快照持有，造成版本链和持久化开销。

### Figure 5：Row vs. Column under OLTP / OLTP 下行列格式对比

![[assets/ByteGraph-Dione_fig5_row_column_oltp.png]]

**说明**：高并发下行存达到约 171 KQPS，列存约 103.3 KQPS，行存优势约 1.65×。

### Figure 6：Row vs. Column under OLAP / OLAP 下行列格式对比

![[assets/ByteGraph-Dione_fig6_row_column_olap.png]]

**说明**：高并发下列存约 110.3 KQPS，行存约 54.5 KQPS，列存优势约 2.02×。

### Figure 7：Traditional Hotspot Replica Issues / 粗粒度副本问题

![[assets/ByteGraph-Dione_fig7_coarse_replica_issue.png]]

**说明**：只为 key 6–8 的热点扩容，却在 RO1/RO2 回放 key 1–8 的全部日志并构页；全量缓存需求进一步降低命中率。

### Figure 8：The Architecture of ByteGraph-Dione / Dione 架构

![[assets/ByteGraph-Dione_fig8_dione_architecture.png]]

**说明**：在原共享存储架构上，事务聚合、自适应双格式和热点数据移动分别解决版本、访问布局和扩缩容问题。

### Figure 9：Lightweight Transactions / 轻量级事务

![[assets/ByteGraph-Dione_fig9_lightweight_transactions.png]]

**说明**：OLAP 按 Group Txn Point 聚合并持久化必要版本；OLTP 独立短版本留在内存并快速回收。

### Figure 10：Row Format / 行格式页面

![[assets/ByteGraph-Dione_fig10_row_format.png]]

**说明**：Pointer Array 指向 RowBlock 中的逻辑版本，rollback pointer 维护历史；compaction 合并存活行。

### Figure 11：Column Format / 列格式页面

![[assets/ByteGraph-Dione_fig11_column_format.png]]

**说明**：ColumnsBlock 以列组织属性，同时保留虚拟行、逻辑指针、commit/delete timestamp 与版本链，支持事务点读。

### Figure 12：Hotspots Scaling Mechanism / 热点扩展机制

![[assets/ByteGraph-Dione_fig12_hotspot_scaling.png]]

**说明**：RO1 只 lazy-apply key 6–8，RO2 只处理 key 8；请求按热点归属路由，使合成场景中两个副本都达到 100% cache hit。

### Figure 13：Overall Performance / 整体性能

![[assets/ByteGraph-Dione_fig13_overall_performance.png]]

**说明**：单机纵向扩核与 2–8 节点横向扩展中，Dione 均显著领先 DB-X/DB-Y 与上一代 ByteGraph；越偏 OLAP，三个新机制的收益越明显。

### Figure 14：OLAP Coalescing / OLAP 聚合效果

![[assets/ByteGraph-Dione_fig14_transaction_coalescing.png]]

**说明**：30/60 秒窗口显著减少 MVCC 版本；相对不聚合的结果差异始终低于 0.00035%。

### Figure 15：Latency under Peak OLAP Load / 峰值分析延迟

![[assets/ByteGraph-Dione_fig15_olap_latency.png]]

**说明**：30 秒窗口延迟最低；60 秒窗口高约 5.48%，不聚合则达到 30 秒方案的 1.338×。

### Figure 16：Transaction Isolation Overhead / 事务隔离开销

![[assets/ByteGraph-Dione_fig16_transaction_overhead.png]]

**说明**：RC 为 97.2 KQPS，传统 SI 为 63.0 KQPS，30 秒聚合 SI 为 81.3 KQPS；聚合显著缩小 SI 与 RC 的差距。

### Figure 17：Storage Format Throughput and Capacity / 格式吞吐与空间

![[assets/ByteGraph-Dione_fig17_format_comparison.png]]

**说明**：双份格式吞吐最高但容量近翻倍；adaptive 接近其吞吐，同时按负载节省约 41.25%–47.25% 空间。

### Figure 18：Page Construction and Cache Hit / 构页与缓存命中

![[assets/ByteGraph-Dione_fig18_page_cache_comparison.png]]

**说明**：细粒度副本平均减少约 98.32% 页面重建；随 follower 从 1 增至 4，命中率从 90% 升至约 93%，旧方案维持约 82%。

### Figure 19：Throughput under Varying Hotspot Ratios / 不同热点比例吞吐

![[assets/ByteGraph-Dione_fig19_hotspot_throughput.png]]

**说明**：热点比例为 10%、40%（正文一处写作 30%）和 80% 时，细粒度方案相对粗粒度提升 3.39%、9.48%、15.65%；热点越集中收益越大。

### Table 1：OLAP/OLTP Workloads

| Workload | Overall Throughput | Query Pattern | Latency | Transaction Isolation |
|----------|-------------------:|---------------|---------|-----------------------|
| OLAP | hundreds of thousands ($10^5$ QPS) | 2+ Hops, PathQuery | up to 30 min (latency insensitive) | Snapshot Isolation |
| OLTP | billions ($10^9$ QPS) | 1 Hop | < 1 s | Read Committed |

**说明**：OLTP 与 OLAP 的规模、查询形态、时延目标和隔离要求完全不同，是分治事务管理的依据。

### Table 2：Real-world Graph Applications

| Applications | Description | OLAP (%) | OLTP (%) |
|--------------|-------------|---------:|---------:|
| Social interaction | Douyin likes, comments and other interactive data | 5–25 | 75–95 |
| Douyin relationship | Douyin relationship data | 10–35 | 65–90 |
| Data lineage | Relationships among offline Hive tables, data tasks and data features | 70–90 | 10–30 |

**说明**：不存在统一固定的 OLTP/OLAP 比例，且业务之间差异极大。

### Table 3：Workload Description

| Workload | OLTP/OLAP Read Query | Read/Write | Description |
|----------|---------------------:|-----------:|-------------|
| More OLTP | 3/1 | 95%/5% | Douyin 关系数据混合负载；OLTP 为一跳 Top-K fan/follow list，OLAP 为多跳关系挖掘/风控 |
| More OLAP | 1/2 | 98%/2% | 同上 |
| OLTP & OLAP equal | 1/1 | 97%/3% | 同上 |
| Micro-benchmark OLAP-only | 100% OLAP Read | 90%/10% | 用于长 OLAP 事务实验 |

**说明**：真实数据包含约 300 万 vertex、5 亿 edge；三种混合比例覆盖日内典型阶段。

---

## 实验

### 数据集与环境

- **数据**：Douyin relationship 生产负载子集，约 3M vertices、500M edges。
- **查询**：OLTP 为单跳精确查询；OLAP 为多跳/path query，服务关系挖掘和风险管理。
- **硬件**：DB-X/DB-Y 使用 8 个 AWS `db.r5.4xlarge` 节点，每节点 16 vCPU、128 GiB DRAM；ByteGraph 系列使用等效内部机器与 append-only 存储。
- **基线**：上一代 ByteGraph、两个匿名商业图数据库 DB-X/DB-Y。GalaxyBase Docker 版因容量与扩展性限制被排除。

### 整体性能

- 相比 DB-X/DB-Y，Dione 吞吐提升约 **20–40×**。
- 相比上一代 ByteGraph：More OLTP 提升约 **35%–50%**；Balanced/More OLAP 提升约 **50%–80%**（正文前后出现 50%–70% 与 50%–80% 两组概述，应以 Figure 13 具体点为准）。
- 单机扩核与多机扩展都保持优势，说明收益不只来自某个固定集群规模。

### 轻量级事务

- 30 秒/60 秒聚合窗口分别减少 **59.44%/64.44%** 历史版本。
- 与无聚合相比，分析结果差异均小于 **0.00035%**。
- 60 秒窗口比 30 秒窗口延迟高约 **5.48%**；无聚合延迟为 30 秒方案的 **1.338×**。
- RC、传统 SI、30 秒聚合 SI 分别为 **97.2/63.0/81.3 KQPS**；SI 相对 RC 的损失从 **35.5%** 降至 **16.49%**。

### 自适应双格式

- 相对 row，adaptive 在 More OLTP/Balanced/More OLAP 下提升 **31.8%/53.5%/70.5%**。
- 相对 column，分别提升 **40%/19.1%/11.7%**。
- 相对 dual，adaptive 仅慢 **2.2%/5.3%/3.7%**，但空间减少 **41.25%/45.75%/47.25%**。
- 列存因保留 OLTP 读加速元数据，相比行存压缩收益约 **30%**，并非纯分析列存的极限压缩率。

### 细粒度热点扩展

- 相对上一代 ByteGraph，重建页面减少约 **98.32%**。
- follower 数 1→4 时，每秒少构建 **1,484/3,069/6,105/13,000** 页。
- Dione cache hit 从 **90%** 升到约 **93%**，上一代约 **82%**。
- 热点流量占比增加时，吞吐收益由 **3.39%** 扩大到 **15.65%**。

---

## 批判性思考

### 优点

1. **机制直接来自生产 trace**：不是泛化地“做 HTAP”，而是针对长短事务、邻接表级访问差异和持续热点分别设计。
2. **粒度选择准确**：格式转换和热点复制都落到 adjacency list/key，而不是表、分区或节点，符合图数据幂律分布。
3. **兼顾成本**：不保留永久双格式，不复制冷页；性能收益与空间、CPU、cache 的节省同时量化。
4. **事务与物理布局协同**：同一 LeafPage 同时考虑 append-only 更新、MVCC、行列扫描和在线转换，工程闭环完整。
5. **实验规模可信**：真实生产衍生 trace、多种比例、单机与集群扩展、模块级消融均有覆盖。

### 局限性

1. **语义不是严格等价**：OLAP 聚合以低于 0.00035% 的结果差换版本缩减，说明它更接近业务可接受的近似快照，而非所有查询都严格逐更新可见；适用边界需明确。
2. **匿名基线不可复现**：DB-X/DB-Y 不具名，外部无法核对配置、公平性或复现实验。
3. **核心底座闭源**：append-only shared storage、完整 ByteGraph 查询层和 trace 均不可用。
4. **成本模型较粗**：把 cache、页位置和 I/O 合并为常数 $C$，在负载骤变、cache residency 改变或远端存储抖动时可能失准。
5. **转换稳定性信息不足**：论文没有公开阈值、采样窗口、迟滞策略、失败恢复和转换中双写的一致性证明。
6. **热点检测细节不足**：未给出检测算法、阈值、自适应副本回收策略，也未量化热点消退后的资源回收成本。
7. **文中数字有轻微不一致**：Figure 19 横轴为 10/40/80%，正文写 10/30/80%；总体 ByteGraph 提升范围也有 50%–70% 与 50%–80% 两种表述。

### 潜在改进方向

1. 用带迟滞的在线学习模型联合预测格式收益、转换成本和热点寿命，减少短期误判。
2. 为聚合 SI 明确定义可见性语义与 error budget，并按查询类型决定是否允许聚合。
3. 在行列成本中显式加入 cache residency、远端 I/O、压缩率和转换摊销成本。
4. 引入热点副本生命周期控制：检测、创建、路由、合并、回收全链路的稳定性与成本上界。
5. 发布匿名化 trace 或 simulator，并使用公开图数据库与同等 SLA 做可复现实验。

### 可复现性评估

- [ ] 代码开源
- [ ] 生产 trace 可获取
- [ ] DB-X/DB-Y 可识别
- [x] 数据规模与负载比例给出
- [x] 硬件规格给出
- [x] 关键实验和消融充分
- [ ] 格式阈值、热点检测和故障恢复细节完整

---

## 关联笔记

### 基于

- [[ByteGraph]]：上一代存算分离图数据库及 BW-Tree forest 基础。
- [[BW-Tree]]：每个 vertex 一棵树、LeafPage 保存邻接表的索引结构。
- [[MVCC]] / [[Snapshot Isolation]]：混合事务版本管理与隔离基础。
- [[Apache Arrow]]：ColumnsBlock 的列式组织参考。
- [[HTAP]]：同一图上统一处理 OLTP 与 OLAP 的系统目标。

### 方法相关

- [[Lightweight Transactions for Hybrid Workloads]]：长 OLAP 聚合与短 OLTP 内存版本。
- [[Adaptive Dual-Format Graph Storage]]：邻接表级 row/column 在线切换。
- [[Row-Column Cost Model]]：在线选择物理格式的简化模型。
- [[Fine-Grained Hotspot Scaling]]：只复制热 key 页面并热点感知路由。
- [[Adjacency List]]：格式选择、热点检测和复制的核心粒度。

---

## 速查卡片

> [!summary] ByteGraph-Dione（SIGMOD Companion 2026）
> - **核心**：面向生产动态图 HTAP，把事务版本、物理格式和热点副本都做到 workload-aware。
> - **事务**：30 秒聚合使历史版本减少 59.44%，SI 吞吐损失从 35.5% 降到 16.49%。
> - **格式**：adaptive 接近 dual 吞吐，只慢 2.2%–5.3%，但节省 41.25%–47.25% 空间。
> - **热点**：少构建 98.32% 页面，cache hit 达 90%–93%，80% 热点比例下吞吐提升 15.65%。
> - **整体**：相对商业基线 20–40×，相对上一代 ByteGraph 约 35%–80%。
> - **代码**：未开源。

---

*笔记创建时间：2026-07-29*
