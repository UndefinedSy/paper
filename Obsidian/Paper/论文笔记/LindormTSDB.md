---
title: "Lindorm TSDB: A Cloud-native Time-series Database for Large-scale Monitoring Systems"
method_name: "LindormTSDB"
authors: [Chunhui Shen, Qianyu Ouyang, Feibo Li, Zhipeng Liu, Longcheng Zhu, Yujie Zou, Qing Su, Tianhuan Yu, Yi Yi, Jianhong Hu, Cen Zheng, Bo Wen, Hanbang Zheng, Lunfan Xu, Sicheng Pan, Bin Wu, Xiao He, Ye Li, Jian Tan, Sheng Wang, Dan Pei, Wei Zhang, Feifei Li]
year: 2023
venue: PVLDB 16(12)
tags: [time-series-database, cloud-native, monitoring, shared-storage, lsm-tree, lindorm]
zotero_collection: VLDB/2023
image_source: local
created: 2026-07-31
---

# 论文笔记：Lindorm TSDB: A Cloud-native Time-series Database for Large-scale Monitoring Systems

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | Alibaba Group、浙江大学、清华大学 |
| 日期 | 2023，PVLDB 16(12): 3715–3727 |
| 项目主页 | — |
| 对比基线 | [[InfluxDB]]、[[TimescaleDB]] |
| 链接 | [DOI](https://doi.org/10.14778/3611540.3611559) |
| 原始 PDF | `VLDB/2023/Lindorm TSDB.pdf` |

---

## 一句话总结

> Lindorm TSDB 以时间与时间线 ID 的二维分片、共享存储、时序专用索引与文件组织、预降采样和下推执行，面向十亿时间线级监控系统实现高吞吐写入、低延迟聚合和弹性扩缩容。

---

## 核心贡献

1. **云原生时序架构**：以 [[Shared-Storage Architecture|共享存储架构]] 将计算节点与 [[LindormDFS]] 解耦，并通过时间范围与时间线 ID 的二维分片避免扩容时搬运历史数据。
2. **时序专用存储与索引**：设计包含 forward/inverted index 与 TSD 的 [[LSM-tree]] 风格引擎，以 [[Roaring Bitmap]]、[[Bloom Filter]]、[[LRU]] `seriescache` 加速高基数标签过滤与聚合。
3. **面向监控查询的执行优化**：用 [[Pre-downsampling|预降采样]]、pipelined execution 和 query pushdown 降低大规模时间线聚合的数据扫描与网络传输。
4. **库内时序 ML**：以 Lindorm ML 通过 SQL 在库内完成训练和推理，避免把原始时序搬运至外部分析系统。

---

## 问题背景

### 要解决的问题

监控系统持续写入带有 metric、timestamp 与多维 tag 的 [[Time Series|时间序列]]。论文中的真实业务 Sys-A / Sys-B 分别拥有至少 **10 亿 / 5 亿** 时间线，日活时间线达 **6 亿 / 4 亿**，写入速率约 **2 亿 / 1.5 亿点每秒**（Table 1）。查询既包括单时间线最新值，也包括命中百万时间线的区域聚合。

### 现有方法的局限

1. **高基数标签索引**：大量时间线的 tag 映射会挤占内存，写入与索引维护相互竞争。
2. **大范围聚合代价高**：标签过滤之后仍可能需要读取、传输、聚合海量原始点。
3. **库外分析链路长**：传统时序 ML 要先导出数据，训练/推理中的 I/O 与网络传输成为瓶颈。
4. **扩缩容迁移重**：本地存储架构通常要迁移历史分片；对追加式监控写入而言，这种迁移不必要且扰动在线服务。

### 本文的动机

监控数据具有明显时间局部性、主要追加写、很少回写历史数据的特性。论文将这些特性编码进分片、文件和 compaction 策略：新节点只接管后续时间段，新产生的数据文件也按时间组织，从而把弹性与历史数据迁移解耦。

---

## 方法详解

### 系统架构

Figure 2 给出论文中的四个主要组件：

- **TSProxy**：接收 OpenTSDB/SQL 等请求、维护元数据视图，并按时间与时间线 ID 将请求路由到 shard。
- **TSCore**：一个节点可管理多个 shard，执行写入、查询、恢复及后台存储任务。
- **Lindorm ML**：在数据库内提供训练和在线推理。
- **[[LindormDFS]]**：共享存储底座，持久化 [[WAL]] 与数据文件；论文描述其兼容 HDFS，并以 ESSD 与 OSS 等存储介质提供热冷层。

每个 shard 由单个 TSCore 独占管理；TSProxy 与 TSCore 均可水平扩展。此处的 `TSCore` 与官网产品架构中“组织 Memchunk、TSFile、TTL/冷热分层”的 TSCore **直接对应**。官网的 `TSCompute`、[[LindormStream]]、[[LindormSearch]] 不在本文 Figure 2 的核心组件内，相关关系见“关联笔记”。

### 模块 1：二维时间分片与弹性

#### Shard group

论文首先按照时间把数据划入连续的 **shard group** $(t_0,t_1]$，每个 group 内再按 timeseries ID 的 hash 分到多个 shard：

$$
\operatorname{shard}(s,t)=\left(\operatorname{group}(t),\ h(\operatorname{id}(s)) \bmod N_{\operatorname{group}(t)}\right)
$$

**含义**：同一时间线在一个时间区间内落在确定 shard；不同时间段可以采用不同数量的 shard。该式为论文 Figure 3 描述的分片规则的形式化记法，并非论文编号公式。

**符号说明**：
- $s$：时间线；$t$：数据点时间戳。
- $\operatorname{group}(t)$：包含 $t$ 的时间范围。
- $h(\operatorname{id}(s))$：时间线 ID 的 hash。
- $N$：此 shard group 的 shard 数。

当集群扩容时，系统为将来的时间范围创建新的 shard group；历史 group 不迁移。该设计依赖监控负载少有历史回写的前提。

### 模块 2：TSM 存储引擎

TSCore 的存储组织是 [[LSM-tree]] 风格：先写内存，再 flush 为有序不可变文件，最后后台 compaction。

#### 写入路径

1. TSProxy 按二维分片规则路由请求。
2. TSCore 先把写请求追加到共享 [[WAL]]，再写入 Memtable。
3. Memtable 到达阈值后 flush，产生 forward index、inverted index 和 TSD 数据文件。
4. 后台 level compaction 合并文件、执行 TTL 清理，并标记可下沉的冷文件。

论文还给出简单 SQL `INSERT` 的 fast path：可绕过 Calcite 的完整解析和优化路径，减少高频写入开销。

#### 文件与索引

| 文件 / 结构 | 键到值 | 用途 |
|---|---|---|
| Forward index | time series ID → tags | 从已知时间线恢复 tag；服务分组和元数据访问 |
| Inverted index | tag key/value → posting list | 根据标签表达式检索匹配的时间线 ID |
| TSD | `(timeseries ID, time range)` → chunks | 保存压缩的时间序列点 |
| Memtable | 最近写入的数据及索引 | 低延迟写入与未 flush 数据读取 |

`seriescache` 只缓存最近使用的“时间线 ID ↔ tags”映射，与缓存文件块的 block cache 分工。两者均采用 [[LRU]]；当原始 tag 很长时，可改缓存其 MD5 值以降低内存占用。

### 模块 3：查询、预降采样与管线执行

典型标签时间范围查询按以下逻辑处理：

1. 利用 inverted index 查找满足标签过滤条件的时间线集合；posting list 采用 [[Roaring Bitmap]]，以快速进行交、并等位图运算。
2. 使用时间范围过滤 TSD 文件，再定位目标 chunk。
3. TSCore 对 shard 内数据执行扫描、sample-by、group-by 与部分聚合；TSProxy 汇总来自多个 shard 的结果。
4. 对常见固定粒度的 downsampling，系统在 flush/compaction 阶段预计算聚合，从而减少查询时读取原始点的次数。

对于 aggregation operator，论文使用 `time-series-wise` 的 pipeline：一个算子可在时间线内流式处理，结果再在 shard/TSCore/TSProxy 层次汇聚。这样对高命中时间线查询，网络中传输的是局部聚合结果而不是全部原始样本。

### 模块 4：Lindorm ML

Lindorm ML 以 SQL 提供模型训练与推理。训练时，TSProxy 将训练请求拆到各 TSCore；各节点基于本地时序数据计算局部结果，并在协调层完成合并。论文以 OneShotSTL 异常检测为例，强调数据库内执行消除了原始数据导出与外部传输。

### 可靠性与恢复

- [[WAL]] 先于 Memtable 持久化，TSCore 故障后其他节点可接管 shard 并回放 WAL。
- shard 与 TSCore 的映射等元数据由 ZooKeeper 管理。
- 论文经验部分描述异步 WAL replay：写服务优先恢复，读服务在回放完成后再开放，缩短故障后可写恢复时间。
- 本文未详细给出 Raft/Paxos 类复制协议、提交点或线性一致性定义；不要把官网其他引擎的多副本语义归因于本文。

---

## 关键公式

论文没有给出编号的数学推导公式。上文的二维分片表达式用于精确说明 Figure 3 的路由规则；其余主要贡献是系统数据路径、索引与执行机制。

---

## 关键图表

> 所有图均由原论文 PDF 高分辨率渲染并存于 `assets/LindormTSDB/`。每个 Figure 的原图与其 caption 在下列页码中完整保留。

### Figure 1：时间序列查询类型（论文 p3）

![[assets/LindormTSDB/LindormTSDB_page3.png]]

**说明**：论文用最新值、单时间线降采样、跨时间线聚合说明监控系统中查询复杂度如何随命中时间线和时间窗口增长。

### Figure 2：Lindorm TSDB 总体架构（论文 p3）

![[assets/LindormTSDB/LindormTSDB_page3.png]]

**说明**：TSProxy、TSCore、Lindorm ML 与 [[LindormDFS]] 四组件；TSProxy/TSCore 可横向扩展。

### Figure 3：二维分片（论文 p4）

![[assets/LindormTSDB/LindormTSDB_page4.png]]

**说明**：按时间切 shard group，组内按时间线 ID hash 分 shard；扩容新建未来 group，不搬运旧 group。

### Figure 4：TSM 存储引擎（论文 p5）

![[assets/LindormTSDB/LindormTSDB_page5.png]]

**说明**：Memtable flush 生成正向/倒排索引与 TSD 文件，后台 compaction 合并并进行生命周期管理。

### Figure 5：写入路径优化（论文 p6）

![[assets/LindormTSDB/LindormTSDB_page6.png]]

**说明**：展示 OpenTSDB/SQL 写入经过 TSProxy、WAL、Memtable 和持久化文件的路径，以及 SQL fast path 的位置。

### Figure 6：SQL 查询总览（论文 p7）

![[assets/LindormTSDB/LindormTSDB_page7.png]]

**说明**：SQL 经解析与优化后进行时间线定位、算子下推和分布式结果合并。

### Figure 7：流水线执行引擎（论文 p8）

![[assets/LindormTSDB/LindormTSDB_page8.png]]

**说明**：operator pipeline 在数据读取与计算之间流式衔接，并支持 shard/TSCore 级局部聚合。

### Figure 8：Lindorm ML 总览（论文 p8）

![[assets/LindormTSDB/LindormTSDB_page8.png]]

**说明**：时序模型通过 SQL 在数据库内部训练、保存与在线推理。

### Figure 9：库内训练流程（论文 p8）

![[assets/LindormTSDB/LindormTSDB_page8.png]]

**说明**：TSProxy 协调多 TSCore 读取本地时序，计算局部训练结果并聚合模型状态。

### Figure 10：不同时间线规模下的写入吞吐（论文 p9）

![[assets/LindormTSDB/LindormTSDB_page9.png]]

**说明**：在最大规模约 100K hosts / 10M timeseries 时，三节点 Lindorm 的写入吞吐相对对比系统约高一个数量级。

### Figure 11：故障与扩容下的写吞吐（论文 p11）

![[assets/LindormTSDB/LindormTSDB_page11.png]]

**说明**：单节点故障后吞吐短暂下降约 4%，约 30 秒恢复；扩容阶段吞吐基本保持平稳。

### Table 1：两个真实监控系统的负载（论文 p1）

| 指标 | Sys-A | Sys-B |
|---|---:|---:|
| 总时间线 | ≥1B | 500M |
| 日活时间线 | 600M | 400M |
| 写入速率 | 200M points/s | 150M points/s |
| 常见查询时间范围 | 1–6h | 1–3h |

### Table 2：Lindorm TSDB 数据模型示例（论文 p3）

**说明**：一条时间线由 metric 与 tags 唯一标识；多个 timestamp/value 样本属于同一时间线。原始示例的列和数据在 p3 渲染页中完整保留。

### Table 3：正向与倒排索引（论文 p5）

| 索引 | 语义 |
|---|---|
| Forward index | time series ID → tag 集合 |
| Inverted index | tag 条件 → 匹配时间线 ID posting list |

### Table 4：三种查询模式（论文 p10）

| 查询 | 语义 |
|---|---|
| Q1 | 每个时间线的最新值 |
| Q2 | 每条时间线在指定窗口内按固定粒度降采样聚合 |
| Q3 | 跨大量时间线、按区域等 tag 分组的聚合 |

### Table 5：Q1 最新值查询延迟（ms，论文 p10）

| Host scale | InfluxDB | Lindorm 1-node | Lindorm 3-node | TimescaleDB 1-node | TimescaleDB 3-node |
|---:|---:|---:|---:|---:|---:|
| 10,000 | 22 | 44 | 53 | 237 | 210 |
| 100,000 | 145 | 90 | 95 | 1530 | 1610 |
| 1,000,000 | 2083 | 464 | 284 | 211689 | 13452 |

### Table 6：Q2 降采样查询延迟（ms，论文 p10）

| Host scale | InfluxDB | Lindorm 1-node | Lindorm 3-node | TimescaleDB 1-node | TimescaleDB 3-node |
|---:|---:|---:|---:|---:|---:|
| 10,000 | 72 | 89 | 91 | 53 | 67 |
| 100,000 | 1046 | 177 | 190 | 502 | 471 |
| 1,000,000 | 15261 | 1165 | 934 | 51916 | 10012 |

### Table 7：Q3 跨时间线聚合延迟（ms，论文 p10）

| Host scale | InfluxDB | Lindorm 1-node | Lindorm 3-node | TimescaleDB 1-node | TimescaleDB 3-node |
|---:|---:|---:|---:|---:|---:|
| 10,000 | 559 | 175 | 164 | 91 | 427 |
| 100,000 | 9437 | 1390 | 809 | 898 | 4296 |
| 1,000,000 | 111815 | 21177 | 6884 | 43630 | 30651 |

### Table 8：时序异常检测效率（秒，论文 p11）

| Host scale | Lindorm ML 训练 | 库外训练 | Lindorm ML 推理 | 库外推理 |
|---:|---:|---:|---:|---:|
| 10,000 | 19.69 | 36.72 | 19.89 | 36.37 |
| 100,000 | 198.53 | 431.66 | 206.11 | 391.89 |

### Table 9：Pushdown 消融（论文 p11）

| Host scale | Q3 with pushdown（ms） | Q3 without pushdown（ms） |
|---:|---:|---:|
| 10,000 | 900 | 2256 |
| 100,000 | 7525 | 25569 |
| 1,000,000 | 94082 | 322840 |

**关键发现**：移除 query pushdown 后，Q3 延迟约增加 2×–3.4×，因为原始数据必须汇集到 proxy 层再完成跨时间线聚合。

### Table 10：seriescache 消融（论文 p11）

| Host scale | 写入吞吐：有 cache（M/s） | 写入吞吐：无 cache（M/s） | Q3 延迟：有 cache（ms） | Q3 延迟：无 cache（ms） |
|---:|---:|---:|---:|---:|
| 1,000 | 5.88 | 4.75 | 160 | 189 |
| 10,000 | 5.28 | 3.60 | 383 | 485 |
| 100,000 | 4.66 | 1.40 | 3549 | 5235 |

**关键发现**：启用 seriescache 使写入吞吐提升 **23.8%–232%**，并让需要 tag 值的 Q3 延迟降低 **15.3%–32.2%**。

### Table 11：不同节点数的写吞吐（M/s，论文 p11）

| Host scale | 2 nodes | 4 nodes | 6 nodes |
|---:|---:|---:|---:|
| 10,000 | 5.05 | 11.55 | 19.78 |
| 100,000 | 5.08 | 11.14 | 19.01 |
| 1,000,000 | 4.64 | 10.99 | 18.06 |

---

## 实验

### 数据集与负载

| 项目 | 内容 |
|---|---|
| 真实负载 | 两个生产监控系统的规模与查询分布，用于动机分析 |
| 合成压测 | [[TSBS]] DevOps workload |
| 写入 batch | 10,000 |
| 查询 | Q1 latest value、Q2 downsampling、Q3 inter-timeseries aggregate |

### 实现细节

- **集群**：5 台阿里云 ECS；4 台数据库机器，每台 16 cores / 64 GB；1 台客户端，32 cores / 128 GB。
- **对比系统**：InfluxDB、TimescaleDB、Lindorm；分别评估单节点与三节点部署。
- **扩展性**：额外评估 2/4/6 节点写入吞吐、节点故障和在线扩容。

### 关键实验结论

1. 最大规模时，三节点 Lindorm 写入吞吐约为其他对比系统的 **10×**。
2. 100 万 hosts 下 Q1 仅 **284 ms**，明显低于 InfluxDB 的 2083 ms 和 TimescaleDB 三节点的 13,452 ms。
3. 库内 OneShotSTL 训练/推理约为库外流程的一半时间，主要节省原始时序数据传输。
4. pushdown 与 seriescache 不是边缘优化：前者显著降低跨时间线聚合延迟，后者同时提升写入与分组查询。
5. 从 2 到 6 节点的吞吐接近线性增长，扩缩容设计避免迁移旧时间段数据。

---

## 批判性思考

### 优点

1. **工作负载贴合**：时间分区与时间线 hash 的组合正中监控“不断产生新数据”的数据分布。
2. **端到端设计**：从索引、存储、查询计划到 ML 都针对减少原始时序扫描和搬运。
3. **弹性设计清晰**：新 shard group 吸收未来写入，避免扩容触发大规模历史迁移。
4. **生产导向验证**：不仅有吞吐和延迟，还测试了节点故障和扩容过程。

### 局限性

1. **负载假设较强**：避免历史迁移依赖“很少回写历史”的监控特征；对大量乱序历史修订的收益可能下降。
2. **查询覆盖有限**：重点是 Q1/Q2/Q3 监控查询，未系统评估复杂 ad-hoc SQL、跨模型 join 或更多分布式 TSDB。
3. **一致性细节不足**：论文强调恢复，但未完整说明副本协议、读写一致性和提交语义。
4. **ML 评估集中**：只用 OneShotSTL 代表库内 ML，其他模型的资源调度与隔离问题未展开。

### 潜在改进方向

1. 为历史回写和极端乱序数据设计自适应 shard/compaction 策略。
2. 由代价模型动态决定预降采样粒度及其存储预算。
3. 展开多副本一致性、跨可用区恢复和延迟分位数的协议级评估。
4. 结合官网的 [[LindormStream]] 与 [[LindormSearch]]，系统评估流式计算和透明索引协同。

### 可复现性评估

- [ ] 代码开源
- [ ] 完整部署脚本
- [x] 系统架构与实验环境描述
- [x] 使用公开 TSBS workload

---

## 关联笔记

### 基于

- [[LSM-tree]]：Memtable → immutable files → compaction 的基础组织。
- [[LindormDFS]]：共享 WAL、索引和 TSD 文件的持久化底座。
- [[WAL]]：故障后未 flush 数据的恢复基础。

### 对比

- [[InfluxDB]]：时序数据库基线。
- [[TimescaleDB]]：基于 PostgreSQL 的时序数据库基线。
- [[LindormUWC]]：同属 Lindorm 的专项引擎，但面向车联网超宽列而非监控时间线。

### 方法相关

- [[Time Series]]：本文的基本数据模型。
- [[Time Series Identifier]]：由 metric 和 tags 标识的时间线。
- [[Inverted Index]]：标签过滤到时间线集合的核心索引。
- [[Roaring Bitmap]]：posting list 的压缩位图实现。
- [[Pre-downsampling]]：预计算多粒度时间窗口聚合。
- [[Query Pushdown]]：在 TSCore 本地执行部分查询算子，减少传输。
- [[Lindorm ML]]：论文中的库内训练与推理组件。
- [[TSBS]]：评测所用 DevOps 基准。

### 产品架构映射（非本文全部直接贡献）

- [[LindormTable]]：官网宽表引擎；不是本文的核心 TSDB 组件。
- [[LindormStream]]：官网称其可为 TSCompute 提供订阅式计算输入；本文未直接将其放入 TSDB 总体架构或实验。
- [[LindormSearch]]：官网称其可作为时序索引存储；本文未把它作为内部依赖展开。

---

## 速查卡片

> [!summary] Lindorm TSDB
> - **核心**: 以时间 + 时间线 ID 的二维分片与共享存储服务十亿级监控时间线。
> - **方法**: TSM 文件、正向/倒排索引、seriescache、RoaringBitmap、预降采样、pipeline + pushdown、库内 ML。
> - **结果**: 最大规模写吞吐约为对比系统 10×；1M hosts latest-value 查询为 284 ms；库内异常检测约节省一半时间。
> - **代码**: 未开源。

---

*笔记创建时间: 2026-07-31*
