---
title: "ByteHouse: ByteDance's Cloud-Native Data Warehouse for Real-Time Multimodal Data Analytics"
method_name: "ByteHouse"
aliases: [ByteHouse, ByteDance ByteHouse]
authors: [Yuxing Han, Yu Lin, Yifeng Dong, Xuanhe Zhou, Xindong Peng, Xinhui Tian, Zhiyuan You, Yingzhong Guo, Xi Chen, Weiping Qu, Tao Meng, Dayue Gao, Haoyu Wang, Liuxi Wei, Huanchen Zhang, Fan Wu]
year: 2026
venue: SIGMOD Companion 2026
tags: [cloud-native-data-warehouse, olap, multimodal-analytics, incremental-processing, vector-search, distributed-cache, learned-query-optimization]
zotero_collection: SIGMOD/2026
image_source: local
created: 2026-08-31
---

# 论文笔记：ByteHouse: ByteDance's Cloud-Native Data Warehouse for Real-Time Multimodal Data Analytics

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | ByteDance、Shanghai Jiao Tong University、Tsinghua University |
| 会议 | SIGMOD Companion '26，2026-05-31 至 06-05，Bengaluru, India |
| DOI | https://doi.org/10.1145/3788853.3803080 |
| 页数 | 14 页（pp. 320–333） |
| 产品 | https://bytehouse.cloud |
| 任务 | 实时 OLAP、增量计算、文本/向量混合检索、多模态分析 |
| 规模 | ByteDance Cloud 400+ 服务；典型生产实验达 100–180 节点 |
| 对比系统 | [[ClickHouse]]、[[StarRocks]]、[[Apache Doris]]、[[Milvus]]、[[pgvector]] |

---

## 一句话总结

> ByteHouse 不是“给 OLAP 引擎外挂向量索引”，而是从文件布局、分布式缓存、三模执行、历史/学习型优化到融合检索纵向协同，把结构化分析、持续增量计算和多模态检索统一进一套云原生共享存储数仓。

---

## 核心贡献

1. **面向多模态的纵向一体化存储**：[[Unified Table Engine]] 以 document–chunk 两级逻辑模型统一结构化字段、文本和向量；stable/delta 段、ByteKV staging 与 Sniffer 自描述列式文件同时兼顾写入、扫描和点查。
2. **弥补存算分离的数据局部性损失**：存储侧 [[CrossCache]] 提供集群共享 SSD 缓存，计算侧 [[NexusFS]] 对齐远端块、磁盘 region 与内存 buffer，形成两级本地化读路径。
3. **统一三种执行语义**：[[Analytic Pipeline Mode|APM]] 服务交互式分析，[[Staged Batch Mode|SBM]] 服务长时 ETL，[[Incremental Processing Mode|IPM]] 通过行级 lineage 和可撤回 delta 算子维护持续结果。
4. **历史与模型驱动的优化器**：在 [[Cascades Optimizer]]、CBO 和 HBO 之上，用回归模型选择谓词下推和 join build side，并用轻量模型在 APM/SBM 间路由。
5. **原生混合检索执行**：通过 cross-table runtime filter、`RANK_FUSION` 与选择性 post-join refinement，把标量约束、关键词和向量检索纳入同一查询计划。
6. **公开+生产负载验证**：ClickBench 总体延迟降低 25.4%；IPM CPU 时间降低 28.4%–69.2%；CrossCache 相比 50% 命中单机缓存在 P50/P90/P99 分别降低约 25%/18%/22%；向量检索吞吐领先 Milvus 50% 以上。

---

## 问题背景

### 要解决的问题

ByteDance 的电商、游戏、金融、日志、知识库与 LLM 服务，需要同时处理表、文本、图片、视频和 embedding。典型查询不再是单纯扫描或单纯 ANN，而是：

- 实时日志/仪表盘上的大规模聚合；
- 新数据持续到达时的增量物化；
- 标量条件约束下的向量搜索；
- 文本 BM25 与向量相似度融合；
- 跨表元数据过滤后再做多模态召回。

### 现有方案的局限

1. **传统数仓**偏大扫描，缺少细粒度增量维护和多模态访问模式感知。
2. **向量数据库或 PostgreSQL 插件**擅长 ANN，但难以全局优化跨表 join、标量过滤与融合排序。
3. **存算分离**虽有弹性，却把热点数据放到远端对象存储，冷读和小块随机 I/O 主导尾延迟。
4. 数据、索引和元数据分文件保存，点查需要多次 open/seek/metadata lookup，引入 I/O amplification。
5. 静态规则/CBO 难以适应不断变化的谓词成本、join cardinality 和资源竞争。

### 本文动机

作者的核心判断是：多模态分析的瓶颈跨越所有层次，局部加一个索引无法解决；必须让**数据布局、缓存、执行模式、优化器和检索算子共享同一套语义与统计信息**。

---

## 系统架构

### Figure 1: ByteHouse 生态与应用场景

![[ByteHouse_fig1_ecosystem.png]]

**说明**：左侧统一接入 table/text/image/video，中央以 unified table engine、distributed cache、incremental processing 和 multi-modal search 支撑 hybrid search/processing/analytics，右侧服务知识库检索、日志分析、文本图片检索和视频去重；已覆盖 ByteDance Cloud 400+ 服务。

### Figure 2: ByteHouse 三层架构

![[ByteHouse_fig2_architecture.png]]

**控制层**：
- Server 完成 parser、analyzer、[[Cascades Optimizer|Cascades+CBO]]、HBO/AI model、分布式调度；
- Catalog Manager 把版本化 schema、partition、index 元数据保存在 ByteKV；
- Global Transaction Manager 发放全局有序 commit timestamp，支持可串行事务与一致快照；
- Daemon Manager 调度 compaction/merge 等后台任务。

**计算层**：无状态 worker 共享一套 optimizer/runtime，在 APM、SBM、IPM 间切换；提供 Min-Max、Set、[[Bloom Filter]] 和多层向量索引，以及 runtime filter、BM25/向量融合等算子。

**存储层**：Unified Table Engine + CrossCache + NexusFS；底层覆盖 TOS/S3/HDFS/local SSD，数据可采用 Sniffer、ORC、Parquet、Lance，计算存储交换基于 [[Apache Arrow]] 以实现零拷贝。

> 架构关键不是单纯“存算分离”，而是共享存储之上重新建立可跨 worker 复用的数据局部性。

---

## 存储层详解

### Unified Table Engine

#### 两级逻辑模型

每个多模态数据集是一张表：
- **document**：文章、图片、视频等逻辑实体；
- **chunk**：语义连贯、可独立 embedding/检索/分析的基本单元；
- 主键为 `(document_id, chunk_id)`；
- schema 同时容纳 structured、nested 和 vector columns。

#### Stable–Delta 物理模型

- **stable segment**：不可变压缩列存，优化扫描吞吐；
- **delta segment**：容纳近期 insert/update/feature refresh；
- [[MVCC]] 在两者间维持一致 snapshot；
- 后台 merge/compaction 渐进地把 delta 合入 stable。

### 公式 1：[[Adaptive Compaction|有界线性 compaction 控制器]]

$$
\alpha = \min\left(1,\max\left(0,k\left(\frac{N_{\Delta}}{N^{*}}-1\right)\right)\right)
$$

**含义**：delta 数量接近平衡点时保守 compaction；积压越多，触发频率、merge batch 和调度优先级线性提高，最终饱和为全强度。

**符号说明**：
- $N_{\Delta}$：活跃 delta segment 数；
- $N^{*}$：平衡状态下的目标 segment 数；
- $k$：对积压的敏感系数；
- $\alpha\in[0,1]$：compaction 强度。

**评价**：控制律简单、可解释、避免振荡，但论文没有报告 $k$、$N^*$ 的选择方法或稳定性/敏感性实验。

### 两阶段写入与点查

1. **ByteKV staging**：小而频繁的 row-level 更新先进入行式 KV，分配 transaction id 并写 WAL；
2. **flush**：达到 size/retention threshold 后转成压缩列段，同时保留 schema evolution 与版本可见性；
3. **point lookup**：在 [[Snapshot Isolation]] 下先查 staging 的最新可见版本，再通过 part-level pruning 查 columnar part。

这是一种典型的 write-optimized + read-optimized 分层，但关键在于两层共享统一事务可见性。

### Sniffer 自描述文件

### Figure 3: Sniffer File Format

![[ByteHouse_fig3_sniffer.png]]

| Region | 内容 | 作用 |
|--------|------|------|
| Data | RecordGroup → ColumnPartition → DataBlock | 压缩类型化列数据；DataBlock 是最小物理单元 |
| Descriptor | Layout Index、Sort-Key Descriptor、Column Statistics、Bloom Filter、Schema Descriptor | 块定位、二分查找、谓词裁剪与解码 |
| Footer | descriptor offsets、file version、CRC | 一次 footer 访问重建布局并校验完整性 |

点查先通过 sort-key descriptor 定位 RecordGroup，再由 layout index 获取精确 DataBlock offset；descriptor 常驻内存，通常只需一次 metadata seek + 一次 block read，论文称达到微秒级。

#### 自适应列编码

Sniffer 对列采样后选择兼顾 footprint 与 decode cost 的编码：
- narrow integer range：Frame-of-Reference + Bitpacking；
- low-cardinality/repetition：RLE；
- categorical string：Dictionary Encoding；
- general/high-entropy text：FSST；
- floating point：ALP。

#### 向量布局

传统 offset/value 扁平数组难以维护 per-vector statistics，也会让稀疏/变长向量产生 padding/offset 开销。Sniffer 使用 [[Length-and-Presence Encoding|Length-and-Presence (L&P)]]：
- 每个向量是独立连续物理单元；
- 可维护 range、norm、nullness；
- 存储量随实际内容而非声明维数增长；
- 向量块可独立使用 FOR/ALP，并适配 SIMD decode。

### CrossCache：存储侧集群缓存

[[CrossCache]] 包含：
- **Cache Coordinator (CC)**：全局 namespace 与 metadata；
- **Cache Node (CN)**：本地 SSD、直接连接后端存储；
- 文件默认切成 **12 MB block**，再细分成 **4 MB chunk**；
- block metadata/placement 由 CN 维护并周期上报 CC；
- non-contiguous chunk 先缓冲再 coalesce；
- 多 CN 把完整 block 并发上传为 temporary object，最后 concat，绕过 HDFS 单文件单 writer 的并发限制。

生产 trace 显示 I/O-intensive workload 超过 70% 时间可耗在 cold read，说明该层不是锦上添花，而是存算分离下的核心补偿机制。

### NexusFS：计算侧本地化

[[NexusFS]] 对齐 remote layout、local SSD region 与 memory buffer：
- **region manager**：本地盘切成通常 1 MB region，再分 segment；全局 offset→segment 索引；FIFO 回收；
- **buffer manager**：segment-aligned 固定池；second-chance replacement；保留数据可直接暴露给执行 pipeline，实现 zero-copy；
- **metadata manager**：两级 hash namespace，inactive entry 可序列化以节省内存。

CrossCache 解决“哪台存储缓存节点有数据”，NexusFS 解决“compute worker 如何以对齐、连续、零拷贝方式消费数据”，二者职责互补。

---

## 计算层详解

### APM：交互式流水执行

[[Analytic Pipeline Mode]] 使用 vectorized + pipeline parallel execution：
- aggregation 根据早期采样的 grouping cardinality/reduction ratio，在 partial aggregation 与 direct shuffle + merge aggregation 间选择；
- join build 端生成 [[Runtime Filter]] 并下推 probe scan；
- credit-based flow control 限制 upstream 生产和 buffer occupancy；
- 若 upstream 已有序，则 ordered consumption 增量 merge，避免全量 materialize + sort。

### SBM：长时批处理

[[Staged Batch Mode]] 面向 ETL 和 LLM 数据规整：
- exchange boundary 切 stage；依赖全部完成后下游 stage 才开始；
- stage 内 task 处理互斥 partition；
- intermediate result 落 temporary storage，充当轻量 checkpoint；
- task failure 只重试 task，不重跑整 stage；
- partition 可分 batch，提高并行度且控制单 task memory footprint。

### IPM：持续增量处理

[[Incremental Processing Mode]] 跨 refresh cycle 保持 operator state，只处理上次 materialization point 之后的 delta。

#### 行级 lineage 与统一 delta 协议

每个 tuple 带两个不可变字段：
- `tuple_key`：跨版本逻辑身份；
- `update_seq`：单调递增版本序。

算子输入输出统一为：

$$
\langle \text{tuple\_key},\text{update\_seq},\text{op}\rangle,
\qquad \text{op}\in\{\text{insert},\text{delete}\}
$$

逻辑 update = 旧版本 delete + 新版本 insert；delete 根据 lineage 找到并撤回物化状态，再向下游传播 delete，实现 compositional retraction。

#### 聚合与 join

- `COUNT/SUM/AVG` 可原生 retract：insert 加、delete 减；
- `MIN/MAX` 删除当前极值时，只在受影响 group 内重算，以额外状态换有界重算；
- inner join 重写为 $\Delta L\Join R$、$L\Join\Delta R$、$\Delta L\Join\Delta R$ 三个并行子查询，再按 lineage reconcile；
- outer join 额外维护 match status，正确撤回/补发 null-extended row。

### 自适应模式与 refresh 控制

APM/SBM selector 从 SQL/plan 提取：query-level、table one-hot access pattern、bottom-up plan structure 三组特征，用轻量 regression 同时预测 latency/CPU/memory，再根据近期 workload percentile threshold 分类；阈值持续重标定，模型周期重训。

IPM 当前仍由用户显式启用，系统只自动控制 refresh interval。

### 公式 2：[[Incremental View Maintenance|滑动窗口平均维护成本]]

$$
T_{\mathrm{avg}}=\frac{1}{N}\sum_{i=1}^{N}T_i
$$

- $T_i$：第 $i$ 个近期 refresh round 的执行时间；
- $N$：窗口大小，通常 3–5。

### 公式 3：[[Refresh Interval Control|下一次刷新间隔]]

$$
\Delta t=\min\left(\max\left(kT_{\mathrm{last}},\Delta t_{\min}\right),\Delta t_{\max}(U)\right)
$$

- $k$：避免过于频繁维护的缩放系数；
- $\Delta t_{\min}$：防 thrashing 下界；
- $\Delta t_{\max}(U)$：随集群利用率变化的上界。

### 公式 4：[[Refresh Interval Control|负载感知上界]]

$$
\Delta t_{\max}(U)=\Delta t_{\mathrm{base}}(1+\alpha U)
$$

高 CPU/I/O 负载时延长 refresh 以降低竞争，低负载时缩短以提高 freshness；$\alpha$ 控制负载敏感度。

---

## 查询优化

### Cascades + CBO

[[Cascades Optimizer]] 在枚举时统一推导 partitioning、sorting、grouping physical properties：
- branch-partitioning top-down algorithm 处理 bushy join tree；
- [[Magic Set Rewriting]] 把 selective subplan 复制到 upstream，提早过滤；
- CTE 根据 contextual reuse/semantics 在 inline、shared、materialized 间做 cost-based 选择。

### HBO 与学习型优化

HBO 将 canonical plan fragment hash 到 history store，复用过去的 selectivity/cardinality。局限是只能命中已见 fragment，因而加入 regression model 泛化到新 query/data distribution。

### Figure 4: 历史执行驱动的智能优化

![[ByteHouse_fig4_ai_optimizer.png]]

#### Predicate Pushdown Selection (PPS)

在远端存储和多模态算子中，“尽量下推”并不总正确：昂贵 UDF/向量条件如果选择性差，会增加 remote I/O/compute 并妨碍后续更有效裁剪。

- WHERE 按顶层 AND 分解为候选；
- 每个 predicate 编为 AST；column/operator one-hot，底向上 pooling 得到结构向量；
- regression 预测 predicate-specific I/O cost；
- runtime 选择成本收益最好的 predicate 下推。

#### Join Side Selection (JSS)

- 对每个 join node 编码左右 subtree；
- binary classifier 预测 left/right build；
- 沿初始 plan 自底向上 refine，交换输入但保留既定 join order；
- 优点是局部修改、上线风险较低；缺点是无法纠正错误 join order。

---

## 多模态查询处理

### 分层向量索引

不同数据新鲜度/成本目标使用不同层：
- 实时层服务最新向量；
- 近实时层在 ingestion 与查询效率间折中；
- 持久层服务稳定大规模数据。

### Figure 5: 混合数据检索示例

![[ByteHouse_fig5_hybrid_search.png]]

### 三类关键优化

1. **Cross-table runtime filtering**：先扫描结构化表得到候选 id，生成 runtime filter 推入向量 scan；既提高标量条件准确性，也在 ANN 前剔除无关行。
2. **Fusion-based hybrid search**：`RANK_FUSION` 融合 BM25 与 vector similarity，可支持加权/归一化排名，避免应用层拼两次查询。
3. **Selective post-join refinement**：先用 hybrid retriever 产生小候选集，再与结构化属性 join 并精排，避免全量跨模态 join。

这三项分别对应 **前过滤、召回融合、后精排**，形成完整 hybrid retrieval pipeline。

---

## 关键图表与实验

> 论文共有 **10 张 Figure、4 个编号公式、0 张正式编号 Table**；本笔记全部保留。论文中的图表实验数据以下亦转写为表，便于检索。

### Figure 6: 标准分析基准

![[ByteHouse_fig6_benchmarks.png]]

#### TPC-DS SF-1000

- 比较 ByteHouse、StarRocks、Doris 的 P50/P75/P95/P99；
- ByteHouse 在所有 percentile 最低；
- P99 中竞品最慢查询包括 Q67/Q78 等，ByteHouse 的最慢查询为 Q78，但总体尾部更低。

> 图只给 log-scale 分布和最慢 query label，论文没有提供逐查询原始数字，因此不应从像素反推虚假精确值。

#### ClickBench Q1–Q43

- ByteHouse 对比 ClickHouse；
- **总体 latency reduction = 25.4%**；
- 多数 query 更快，但图中也存在个别 query 接近或略逊，说明不是逐查询绝对支配。

### Figure 7: IPM 效率

![[ByteHouse_fig7_incremental.png]]

#### Inner join（TPC-H SF-100，update ratio 2.5%）

| Query | Full CPU Time (s) | Incremental CPU Time (s) | 降幅 |
|------|-------------------:|-------------------------:|-----:|
| Q12 | 10.9 | 7.8 | 28.4% |
| Q14 | 12.0 | 3.7 | 69.2% |
| Q19 | 23.0 | 13.0 | 43.5% |

#### Q12 不同更新比例

| 更新表 | Ratio | Full (s) | Incremental (s) | 降幅 |
|--------|------:|---------:|----------------:|-----:|
| orders | 2.5% | 71.9 | 54.0 | 24.9% |
| lineitem | 2.5% | 66.2 | 53.9 | 18.6%（正文概括约 19.9%） |
| orders | 5% | 95.3 | 66.2 | 30.5% |
| lineitem | 5% | 119.7 | 77.2 | 35.5% |
| orders | 10% | 142.4 | 86.0 | 39.6% |
| lineitem | 10% | 349.9 | 134.0 | 61.7% |

**观察**：IPM 并非只在 delta 极小时有效；在该 join-only workload 中，更新比例增大时全量重算增长更快，增量优势反而扩大。

### Figure 8: CrossCache latency

![[ByteHouse_fig8_crosscache.png]]

**环境**：180 compute nodes；每节点 60 CPU cores、256 GB RAM、2.5 TB SSD；CrossCache 为 1 CC + 47 CN，每 CN 2.25 TB SSD；workload 是 scan volume 最大的 1000 条历史查询。

| Percentile | No Cache | Single 100% | Single 50% | CrossCache |
|------------|---------:|------------:|-----------:|-----------:|
| P50 | 2.5 s | 1.4 s | 2.0 s | 1.5 s |
| P75 | 3.8 s | 1.7 s | 2.9 s | 2.1 s |
| P90 | 4.4 s | 2.2 s | 3.3 s | 2.7 s |
| P95 | 4.7 s | 2.6 s | 3.9 s | 3.0 s |
| P99 | 5.6 s | 2.9 s | 4.5 s | 3.5 s |

CrossCache 不如“理想 100% 命中单节点”，但显著优于更现实的 50% 单节点命中；核心价值是跨 compute 调度后仍可复用缓存。

### Figure 9: AI-driven optimization

![[ByteHouse_fig9_ai_optimization.png]]

**环境**：100 节点，每节点 192 CPU cores、1.48 TiB RAM。

- PPS 在 T+3 开启后，average read volume **最多下降 87%**，≥10 s timeout rate 与平均 latency 同步显著下降；
- JSS 使用 1000 条生产 join query；P50–P99 latency 相比 baseline **降低 15%–45%**，P95/P99 尾部收益最大；
- 论文没有随机对照、置信区间、模型训练开销与回滚率，生产 before/after 结果可能混入 workload drift。

### Figure 10: 多模态查询性能

![[ByteHouse_fig10_multimodal.png]]

#### Vector throughput（VDBBench，99% recall，1% scalar filter）

| Dataset | ByteHouse | Milvus | pgvector |
|---------|----------:|-------:|---------:|
| Cohere | 2233 QPS | 1473 QPS | 2075 QPS |
| C4 | 1186 QPS | 659 QPS | 1150 QPS |

- 相比 Milvus：Cohere +51.6%，C4 +80.0%；
- 相比 pgvector：Cohere +7.6%，C4 +3.1%；
- 因此“领先 50%+”特指 Milvus，不应泛化为对所有 baseline。

#### Hybrid recall

- MS MARCO 含 8.84M passages，实验抽样 2M；embedding 为 [[BGE-M3]]；
- hybrid 在 Recall@5/10/50/100 均最好；
- Recall@100 相比 vector search 约 +30%，相比 text search 约 +50%；
- 图展示的是 normalized recall，未报告绝对 recall、索引构建时间和 latency–recall 曲线。

---

## Lessons Learned

1. **durability 与 availability 解耦**：write-through 简单但 cold miss 慢，write-back 隐藏写延迟却复杂；应让持久化异步进行而不中断服务。
2. **混合布局仍有价值**：hot tuple 用 row format，cold region 用 compressed columnar format，兼顾更新与扫描带宽。
3. **高并发必须端到端协同**：$10^4$–$10^5$ read/write QPS 下，prepared statement、TopN short-circuit、CrossCache 批量细粒度访问共同减少控制与 I/O amplification。
4. **向量计算只是起点**：生产 workload 正从 simple top-k 走向 scalar filter、similarity-bounded aggregation、vector-driven word cloud、multi-vector join recall，需要向量与传统 relational operator 深度融合。

---

## 批判性思考

### 优点

1. **真正端到端**：论文最强之处不是某个单点算法，而是把 document/chunk、Sniffer、CrossCache/NexusFS、三模执行、optimizer 与 hybrid retrieval 串成完整设计。
2. **工程边界清晰**：CrossCache 与 NexusFS、APM 与 SBM、HBO 与 regression model 各自解决不同层级问题，没有把所有东西塞进一个“智能模块”。
3. **IPM 语义较完整**：不仅说“做增量”，还解释 update 的 delete+insert、aggregate retraction、MIN/MAX fallback、inner/outer join correction。
4. **对 learned optimizer 的上线策略务实**：PPS 预测 I/O cost，JSS 只局部交换 build side；相比端到端 plan generation 更容易约束风险。
5. **证据来自多层次**：标准 benchmark、微型机制评估、真实生产 trace 都有覆盖。

### 局限性

1. **系统论文广而不深**：每个组件都可单独成文，但大量关键参数未公开：compaction controller、cache consistency/failure recovery、tiered vector index、model architecture/feature dimension/training cadence。
2. **缺少公平的端到端竞品配置**：TPC-DS/ClickBench 图未列硬件、版本、参数、成本；无法判断优势来自算法、配置还是资源。
3. **多模态 baseline 解读容易夸大**：ByteHouse 对 Milvus 提升显著，但对 pgvector 仅约 3%–8%；摘要“50% higher throughput”有选择性。
4. **学习型优化证据不够因果**：生产逐日曲线无 A/B 隔离、误差条与 workload composition 控制；未报告误判导致 regression 的比例。
5. **CrossCache 只比较单节点 cache**：缺少 Alluxio/Databricks IO cache 等分布式缓存 baseline，也未报告一致性、重平衡、元数据 coordinator 压力和故障恢复。
6. **IPM 覆盖范围不清**：只评估 inner join/Q12 更新；没有复杂 DAG、outer join correction、non-retractable aggregate、skew 和 state growth 的实验。
7. **Sniffer 声称微秒点查但无独立实验**：编码选择、L&P、footer 自描述、point lookup 都缺少 ablation。
8. **事务语义论述过简**：GTM 被描述为提供 serializable transaction，但论文没有协议、冲突检测、failover 和扩展性证据。
9. **成本维度缺失**：云原生系统应同时报告性能/美元、缓存 SSD 成本、对象存储请求成本和模型维护成本。

### 潜在改进方向

1. 对 Sniffer、L&P、NexusFS alignment、CrossCache chunk size 做逐项 ablation。
2. 用 shadow optimizer/online guardrail 报告 PPS/JSS 的 p50 gain、worst regression、fallback rate 与模型漂移。
3. 对 IPM 增加 state-size、skew、outer join、MIN/MAX 删除和长时运行实验。
4. 与分布式 cache 及现代 lakehouse/data warehouse 做同成本对比。
5. 公布 hybrid query 的 latency–recall–QPS 三维 Pareto curve，而非只固定 99% recall。
6. 说明 GTM 的高可用和 timestamp bottleneck，避免“serializable”停留在架构图标签。

### 可复现性评估

- [ ] 系统代码开源
- [ ] Sniffer/CrossCache/NexusFS 实现公开
- [ ] 模型、特征与训练数据公开
- [x] 公开 benchmark 名称明确
- [ ] 公开 benchmark 的完整硬件/版本/配置充分
- [x] 多项生产规模与关键数字给出
- [ ] 原始结果与统计显著性公开

> **总评**：这是一篇优秀的工业系统“全景设计说明”，最值得学习的是**纵向协同与生产化取舍**。但它更像 ByteHouse 2026 年能力地图，而不是对每个新机制都完成严密因果验证的学术论文。读设计可以高信任，读性能数字则必须限定 workload、baseline 与未披露配置。

---

## 关联笔记

### 核心架构

- [[Shared-Storage Architecture]]：云原生弹性基础与远端 I/O 问题来源
- [[Unified Table Engine]]：统一结构化、文本和向量数据
- [[Sniffer File Format]]：自描述列式布局
- [[CrossCache]]：存储侧共享 SSD cache
- [[NexusFS]]：计算侧 alignment-aware local path
- [[MVCC]] / [[Snapshot Isolation]]：stable/delta 与 staging/columnar 的一致可见性

### 执行与优化

- [[Analytic Pipeline Mode]]：交互式流水分析
- [[Staged Batch Mode]]：可恢复批处理
- [[Incremental Processing Mode]]：持续增量维护
- [[Incremental View Maintenance]]：IPM 的理论类别
- [[Cascades Optimizer]]：搜索框架
- [[History-Based Optimization]]：历史执行统计复用
- [[Predicate Pushdown Selection]]：学习型下推选择
- [[Join Side Selection]]：学习型 build/probe 决策
- [[Runtime Filter]]：join 与 vector scan 提前过滤

### 多模态与基准

- [[Hybrid Search]]：文本+向量融合检索
- [[Milvus]] / [[pgvector]]：向量基线
- [[ClickHouse]] / [[StarRocks]] / [[Apache Doris]]：分析基线
- [[TPC-DS]] / [[ClickBench]] / [[TPC-H]]：OLAP benchmark
- [[VDBBench]] / [[MS MARCO]] / [[BGE-M3]]：向量与检索实验

---

## 速查卡片

> [!summary] ByteHouse (SIGMOD Companion '26)
> - **定位**：实时多模态云原生共享存储数仓
> - **存储**：document–chunk + stable/delta + ByteKV staging + Sniffer
> - **缓存**：CrossCache（12 MB block / 4 MB chunk）+ NexusFS（1 MB region、aligned zero-copy）
> - **执行**：APM 交互 / SBM 批处理 / IPM 增量
> - **优化**：Cascades+CBO+HBO，ML 做 PPS/JSS 与 APM/SBM 选择
> - **检索**：cross-table runtime filter + RANK_FUSION + post-join refinement
> - **结果**：ClickBench −25.4%；IPM CPU −28.4%～69.2%；CrossCache P99 4.5→3.5 s；对 Milvus QPS +51.6%/+80.0%
> - **警惕**：对 pgvector 只领先约 3%–8%；大量机制缺 ablation、配置与开源实现

---

*笔记创建时间：2026-08-31 20:58（Asia/Shanghai）*
