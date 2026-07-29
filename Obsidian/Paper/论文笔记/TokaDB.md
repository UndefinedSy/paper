---
title: "TokaDB: A Unified Storage Engine for Training-Serving Data Management in Large Recommendation Models"
method_name: "TokaDB"
authors: [Peng Fang, Kelei Guo, Cheng Chen, Wei Zhang, Mingming Chen, Huaye Xu, Borong Meng, Zongjia Chen, Mingshuai Wang, Luping Wang, Yuan Zhang, Shiru Ren, Fang Wang, Dan Feng]
year: 2026
venue: SIGMOD-Companion
tags: [storage-engine, large-recommendation-model, training-serving, kv-cache, hybrid-access, resource-isolation, disaggregated-storage]
zotero_collection: SIGMOD/2026
image_source: local
created: 2026-07-28
---

# 论文笔记：TokaDB

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | Huazhong University of Science and Technology、ByteDance Inc. |
| 日期 | May 31–June 5, 2026 |
| 项目主页 | - |
| 对比基线 | [[HBase]]、Apache Doris、Doris+ |
| 链接 | [DOI](https://doi.org/10.1145/3788853.3803078) |

---

## 一句话总结

> TokaDB 用一套存储引擎统一承载大推荐模型的训练与在线服务，通过混合访问数据组织、HDD 协同 I/O 和优先级资源管理，在真实生产负载上实现 4–10× 端到端加速。

---

## 核心贡献

1. **Hybrid-Access Specialized Data Organization**：以 [[Detached-Segment Index Layout]] 同时服务用户行为序列的高吞吐点读与高带宽范围扫描，并通过 [[Locality-Triggered Garbage Collection]] 和 [[Hot-Cold Tiering]] 恢复局部性、降低 SSD 成本。
2. **Cost-Efficient Cooperative I/O Engine**：针对超大规模 [[KV Cache]] 的 HDD 部署，联合 [[Adaptive Backup Reads]]、[[Hardware-Aware Block Alignment]] 和 [[Log as Data]]，兼顾尾延迟与写吞吐。
3. **Priority-Based Resource Manager**：用 [[Preemptive Thread Scheduler]]、[[Hierarchical Cache Allocator]] 和 [[Multi-Layer Integrated Rate Limiter]] 在 CPU、缓存和 I/O 三个层面保护高优先级在线服务。
4. **统一训练—服务数据底座**：同时管理 [[User Behavior Sequence|用户行为序列]] 与 KV Cache，支持 Table、SQL、Vector 等接口，并运行在可插拔的 [[Disaggregated Storage-Compute Architecture|存算分离架构]] 上。
5. **生产级验证**：在字节跳动电商与广告真实负载上，相比 HBase、Doris、Doris+ 获得 4–10× 端到端加速、2–3× 成本效率，并满足 LRM 的五维性能目标。

---

## 问题背景

### 要解决的问题

[[Large Recommendation Model|大推荐模型（LRM）]]把 Transformer 引入推荐系统后，训练—服务生命周期产生两类截然不同的数据和四种关键访问：

- **用户行为序列**：在线 serving / 流式训练需要高吞吐 Point Read；离线训练需要高带宽 Range Scan。
- **KV Cache 数据**：在线 serving 需要低 P99 随机读；训练过程持续生成大量新模型 Cache，需要高写带宽。
- **优先级冲突**：在线 serving 必须优先于训练和 compaction / GC 等后台活动，但它们共享 CPU、缓存和存储带宽。
- **成本约束**：单模型 KV Cache 可达数十 TB，数百个模型并行产生 EB 级容量，纯 SSD 部署成本不可接受。

### 现有方法的局限

- **行存 / HBase**：同一用户记录物理连续，点读较好；但无法按时间精确过滤，读最近一段记录时产生高读放大，范围扫描带宽不足。
- **列存 / Doris**：时间分区利于范围过滤，但同一用户序列散落在多个分区，需要跨分区 join 才能拼成模型输入。
- **列优化 Doris+**：缓解 join 与范围扫描问题，但索引和数据仍耦合，难以同时优化两种访问。
- **HDD KV 存储**：随机读 IOPS 低，慢盘、后台 compaction 会放大尾延迟；WAL 与数据文件双写又挤占训练写带宽。
- **通用资源控制**：无法识别 serving / training 优先级，低优先级突发会抢占 CPU、驱逐 serving 热数据并占满 I/O。

### 本文的动机

LRM 的两个数据域并非两个孤立系统：它们共同参与训练和 serving，共享相同用户与模型生命周期。分拆部署会增加运维复杂度、形成资源孤岛；统一引擎只有同时解决**混合访问、成本—性能、优先级隔离**三个问题才有意义。

---

## 方法详解

### 系统架构概览

TokaDB 分为三层：

1. **Hybrid-Access Specialized Data Organization**：管理用户行为序列的数据布局、局部性修复和 SSD/HDD 分层。
2. **Cost-Efficient Cooperative I/O Engine**：管理 KV Cache 的读尾延迟、物理块布局与写放大。
3. **Priority-Based Resource Manager**：协调 CPU 线程、缓存空间和设备带宽。

系统进一步采用 [[Disaggregated Storage-Compute Architecture|存算分离]]：Compute Node 承担表引擎和资源管理，底层通过统一 SDK 接入 ByteStore 或 HDFS；Cluster Manager 负责成员关系、namespace、partition placement、failover、负载均衡与 RBAC；上层暴露 Table、SQL、Vector 等插件式接口，并可直接读写 Parquet 以连接 Spark / Flink 数据湖生态。

### 核心模块

#### 模块 1：[[Detached-Segment Index Layout]]

**设计动机**：既保留“同一用户数据聚合”的点读局部性，又获得“按时间段精确过滤”的范围扫描能力。

**两步转换**：

1. 以 user ID 为 key，把该用户所有按时间排序的行为记录聚合成一个大 value。
2. 将大 value 按时间切为多个 segment，顺序写入独立 [[ValueLog]]；每个 segment 的地址、时间范围、记录数写入紧凑的 Index [[SSTable]]。

**读路径**：

- Point Read：先读 Index SSTable，按所需长度 / 时间定位 segment，再从 ValueLog 读取；无需列存的跨分区 join。
- Range Scan：遍历 Index SSTable 中目标用户，只提取满足时间范围或最大条数的 segment；例如读取最近 1,000 / 总计 10,000 条记录可减少最多 90% 读放大。

#### 模块 2：[[Locality-Triggered Garbage Collection]]（LTGC）

Index SSTable compaction 会让同一用户的 segment 分散到多个 ValueLog。传统 BlobDB GC 只回收失效空间，无法修复“用户级碎片”。LTGC 在 compaction 时统计：

- $LS_k$：key $k$ 的数据分布在多少个不同 ValueLog segment 中；
- $\overline{LS}$：所有 key 的平均 locality score；
- $F_{LS}$：$LS_k$ 超过高阈值 $\tau$ 的 key 比例。

命中触发条件后，LTGC 把同一用户散落的 segment 聚合写入新 ValueLog。它以局部性而非过期数据量作为触发依据，在 GC I/O 与局部性之间建立可调节平衡。

#### 模块 3：[[Hot-Cold Tiering]]

生产数据中，最不活跃的 40% 用户行为 segment 只贡献不到 10% 读取。TokaDB 因而维护 SSD 与 HDD 两类 ValueLog，并为每个用户配置 $F_{HDD}$：

- $F_{HDD}=0.4$ 表示最老的 40% 行为记录放 HDD，最新 60% 留在 SSD。
- GC 期间根据当前分布与目标比例迁移 segment，使每个用户的冷数据比例逼近配置值。
- 实验显示 $F_{HDD} \le 0.7$ 时读取性能基本稳定；更激进的 HDD 比例开始损伤性能。

#### 模块 4：[[Adaptive Backup Reads]]

首次副本在 $T_{backup}$ 内未完成时，并行读取跨机架的另一副本；接受第一个成功响应并立即取消其余请求。静态 deadline 会在拥塞时触发“更多备份读 → 更拥塞”的正反馈，因此 TokaDB 每 100 ms 根据设备 queue depth、吞吐、利用率和延迟计算归一化压力 $S_t \in [0,1]$，闭环调整 deadline：

- $S_t>S_0$：增大 deadline，抑制新 backup read；
- $S_t<S_0$：减小 deadline，更积极地隐藏慢副本；
- 典型参数：$\eta=20$ ms、$S_0=0.7$，CPU 开销低于 1%。

#### 模块 5：[[Hardware-Aware Block Alignment]]

ByteStore 使用固定 2 MB 物理块，但逻辑相邻块未必物理相邻；线上超过 90% 的逻辑相邻块对物理不连续。未对齐的大 KV 会跨块，平均 seek 数最多增至 3×，有效吞吐下降 29%。

TokaDB 将一个 KV Cache entry 完整映射到一个物理块：块内依次存 metadata、data 和 padding。少量、上界明确的空间放大换取单次顺序 I/O 与稳定 P99。

#### 模块 6：[[Log as Data]]

传统 [[WAL]] + SSTable 会把同一 value 持久化两次。TokaDB 直接将 log 作为主数据：修改 [[LSM-tree]]，SSTable entry 只保存指向 log data block 的轻量指针，读取时先查指针再直读 log。底层存储仍复制 log，因此 durability、crash consistency 和容错语义不变，但消除了第二次数据写入。

#### 模块 7：[[Preemptive Thread Scheduler]]

每个 CPU core 维护高优先级队列 HPQ 与低优先级队列 LPQ：

- HPQ：serving 的用户序列点读与 KV Cache 读取；
- LPQ：training、compaction、GC 等后台工作。

HPQ 非空时优先调度；高优请求到达可在用户态抢占低优任务，异步 I/O 完成后立即恢复并再次触发抢占。为避免 training 永久饥饿，少量 core（例如 64 核中的 8 核）专供 LPQ。该设计使上下文切换低于 1 ms，并将 P99 约降低 25%。

#### 模块 8：[[Hierarchical Cache Allocator]]

缓存按优先级分三层：

1. **Metadata Pool**：最高优先级，预热并 pin [[Bloom Filter|filter block]]、Index SSTable 等关键元数据，避免 KV separation 的 double-read。
2. **Serving Data Pool**：通常占总缓存 50% 以上，用 [[LRU]] 缓存 serving 热用户行为与 serving 模型 KV Cache。
3. **Staging Pool**：剩余容量服务 training 与后台活动，采用 FIFO，利用短期局部性但不污染 serving cache。

三层分配使吞吐提高约 63%，P99 下降约 13%。

#### 模块 9：[[Multi-Layer Integrated Rate Limiter]]

- **Request-Layer Throttler**：按 serving > training / background 的优先级发 token。
- **Hardware-Layer Throttler**：每个物理 I/O 也需 token；Sentinel 监控设备利用率和队列深度，在高水位时降低低优流量 token rate，对高优 serving 保持放行。

双层设计避免“同类请求实际 I/O 差异大”与“物理层盲目限流误伤 serving”两个问题。

---

## 关键公式

### 公式 1：[[Locality-Triggered Garbage Collection|LTGC 触发条件]]

$$
\overline{LS} > \theta
\quad \text{or} \quad
F_{LS} > \gamma
$$

**含义**：平均碎片度过高，或高度碎片化 key 的比例过高时触发局部性修复。

**符号说明**：
- $LS_k$：key $k$ 横跨的 ValueLog segment 数。
- $\overline{LS}$：参与 compaction 的所有 key 的平均 locality score。
- $F_{LS}$：满足 $LS_k>\tau$ 的 key 比例。
- $\theta$：平均局部性阈值，示例为 3。
- $\tau$：高局部性阈值，示例为 5。
- $\gamma$：高度碎片化 key 的最小比例，示例为 0.3。

### 公式 2：[[Adaptive Backup Reads|Backup Read Deadline 更新]]

$$
T_{backup}(t+1)
=
T_{backup}(t)
+
\eta\bigl(S_t-S_0\bigr)
$$

**含义**：根据设备压力与目标压力的偏差闭环调节备份读 deadline，在尾延迟优化和额外 I/O 之间自适应平衡。

**符号说明**：
- $T_{backup}(t)$：采样时刻 $t$ 使用的 backup deadline。
- $S_t \in [0,1]$：由设备队列、吞吐、利用率、延迟加权得到的资源压力。
- $S_0$：目标压力，典型值 0.7。
- $\eta$：调整步长，典型值 20 ms。
- 采样间隔 $\Delta$：100 ms。

### 公式 3：[[Hardware-Aware Block Alignment|块内 Padding 大小]]

$$
PaddingSize
=
BlockSize
-
\bigl(MetaSize + DataSize\bigr)
$$

**含义**：用 padding 填满 metadata 与 data 后的块内剩余空间，使每条 KV Cache entry 与物理块边界一一对齐。

**符号说明**：
- $BlockSize$：底层固定物理块大小，ByteStore 中通常为 2 MB。
- $MetaSize$：entry 元数据大小。
- $DataSize$：KV Cache value 大小。
- $PaddingSize$：为对齐而填充的空间。

---

## 关键图表

### Figure 1：The LRM workflow and the objective of TokaDB / LRM 工作流与目标

![[assets/TokaDB_fig1_workflow.png]]

**说明**：LRM 同时包含在线 serving、流式训练与离线训练；对应写带宽、点读吞吐、读延迟、范围扫描带宽和成本五个评价维度。TokaDB 目标是在所有维度同时达到生产要求。

### Figure 2：Two primary workloads in LRM / 两类核心数据负载

![[assets/TokaDB_fig2_workloads.png]]

**说明**：用户行为序列被多个模型共享，KV Cache 则由每个模型单独生成；前者需要 point read + range scan，后者需要随机 read + 持续 write。

### Figure 3：Storage structures for user behavior sequences / 用户行为序列存储布局

![[assets/TokaDB_fig3_storage_structures.png]]

**说明**：对比 row store、column store、column-optimized store。三者分别在时间过滤、用户聚合和 join 开销之间取舍，没有一种原生同时满足两类访问。

### Figure 4：Performance on user behavior sequences / 用户行为序列性能

![[assets/TokaDB_fig4_sequence_performance.png]]

**说明**：HBase 点读相对较好但 range scan 远低于 2.75 TB/s 需求；Doris / Doris+ 受跨分区组织和 join 影响，不能同时达到点读与扫描目标。

### Figure 5：Performance on KV Cache data / KV Cache 性能

![[assets/TokaDB_fig5_kv_cache_performance.png]]

**说明**：在 HDD 三副本、20M 条、每条约 1.6 MB 的条件下，HBase / Doris / Doris+ 无法同时满足 800 QPS、P99 < 100 ms、写入 ≥ 250 MB/s。

### Figure 6：System overview of TokaDB / 系统架构

![[assets/TokaDB_fig6_system_overview.png]]

**说明**：展示三项核心技术、Cluster Manager、多接口层与 ByteStore / HDFS 可插拔存储底座的整体关系。

### Figure 7：Detached-Segment Index Layout / 索引—段分离布局

![[assets/TokaDB_fig7_detached_segment_layout.png]]

**说明**：先按用户聚合，再按时间切 segment；Index SSTable 保存地址、时间范围和条数，ValueLog 保存完整行为数据。

### Figure 8：Cost-Efficient Cooperative I/O Engine / 成本高效协同 I/O 引擎

![[assets/TokaDB_fig8_cooperative_io.png]]

**说明**：三项优化分别作用于慢副本尾延迟、跨块随机 seek 和 WAL / data file 双写，共同改善 HDD 上的读写竞争。

### Figure 9：Priority-Based Resource Manager / 优先级资源管理器

![[assets/TokaDB_fig9_resource_manager.png]]

**说明**：用户态抢占调度管理 CPU，三级 cache pool 隔离缓存，request / hardware 双层 throttler 管理 I/O；三者共享 serving 高于 training 的统一优先级语义。

### Figure 10：End-to-end running time / 端到端运行时间

![[assets/TokaDB_fig10_end_to_end.png]]

**说明**：电商场景 TokaDB on ByteStore 为 65 min，广告场景 77 min，是唯一在 100 min 窗口内完成全部负载的系统；相比 HBase / Doris 分别约快 5× / 3–4×。

### Figure 11：Point Read Throughput and Range Scan Bandwidth / 混合访问性能

![[assets/TokaDB_fig11_hybrid_access_results.png]]

**说明**：在 5K、10K、20K 和 Mix 序列长度下，TokaDB 点读最高领先 HBase 1.8×、Doris 1.6×、Doris+ 1.3×；range scan 最高领先 HBase 3.6×、Doris 2.5×、Doris+ 1.7×。

### Figure 12：P99 read latency and max write bandwidth / KV Cache 读写性能

![[assets/TokaDB_fig12_kv_cache_results.png]]

**说明**：TokaDB 在 P99 < 100 ms 下支持最多 30% 更高 QPS，同吞吐 P99 最多下降 70%；在 85% 最大读负载下，写带宽领先 HBase 52%、领先 Doris / Doris+ 60%。

### Figure 13：Parameter sensitivity for Hot-Cold Tiering / 冷热分层参数敏感性

![[assets/TokaDB_fig13_tiering_sensitivity.png]]

**说明**：两种工作负载在较小 $F_{HDD}$ 下读吞吐稳定；约从 0.7 后明显下降。该参数允许运维者显式交换 SSD 成本与读取性能。

### Figure 14：Ablation of Priority-Based Resource Manager / 资源管理器消融

![[assets/TokaDB_fig14_resource_manager_ablation.png]]

**说明**：去掉抢占调度主要恶化 P99；去掉分层缓存同时损伤吞吐与 P99；去掉双层限流使低优先级 I/O 干扰 serving，广告高负载下影响更明显。

### Table 1：并发 serving 与 training 时的性能退化

| Metrics | HBase | Doris | Doris+ |
|---------|------:|------:|-------:|
| Point Read Throughput | -65% | -28% | -35% |
| Range Scan Bandwidth | -18% | -27% | -10% |
| P99 Read Latency | +40% | +78% | +81% |
| Write Bandwidth | -50% | -27% | -23% |

**说明**：所有系统都出现明显资源争用；低优先级 training 与后台任务占用 CPU、I/O 和共享 block cache，直接破坏高优 serving。

### Table 2：评测负载统计

| 数据类型 | Metric | E-commerce | Advertising |
|---------|--------|-----------:|------------:|
| User Behavior Sequences | Dataset Total Size | 24.52 TB | 37.11 TB |
| User Behavior Sequences | Average Length | 15.32K | 24.28K |
| KV Cache Data | KV Entry Size | 1.6 MB | 1.6 MB |
| KV Cache Data | Dataset Total Size | 122 TB | 167 TB |

**说明**：每个场景覆盖 1,000 万用户和 20 个模型；广告的活跃用户与请求更多，因此行为序列更长、KV Cache 总量更大。

### Table 3：TokaDB 与基线成本对比

#### (a) E-commerce

| System | Compute Nodes | SSDs | HDDs | Total Cost (USD/day) |
|--------|--------------:|-----:|-----:|---------------------:|
| HBase | 12 | 38 | 197 | 1411.01 |
| Doris | 18 | 37 | 207 | 1736.90 |
| Doris+ | 8 | 24 | 167 | 1039.65 |
| **TokaDB** | **6** | **24** | **72** | **655.02** |

#### (b) Advertising

| System | Compute Nodes | SSDs | HDDs | Total Cost (USD/day) |
|--------|--------------:|-----:|-----:|---------------------:|
| HBase | 17 | 49 | 314 | 2075.54 |
| Doris | 29 | 54 | 302 | 2671.89 |
| Doris+ | 12 | 24 | 262 | 1524.86 |
| **TokaDB** | **6** | **24** | **72** | **655.02** |

**说明**：按火山引擎公开价格计算，TokaDB 的整体成本效率为基线的 2–3×。而且基线即使扩容也不能达到 100 ms P99，成本比较已把它们的目标放宽到 160 ms。

### Table 4：LTGC 性能指标

#### (a) E-commerce

| GC Method | GC I/O | Avg Locality | High-locality Ratio |
|-----------|-------:|-------------:|--------------------:|
| BlobDB GC | 170.8 TB | 6.2 | 11% |
| LTGC $\theta=2,\tau=4$ | 122.2 TB | 2.1 | 3% |
| LTGC $\theta=3,\tau=5$ | 77.3 TB | 2.7 | 6% |
| LTGC $\theta=4,\tau=6$ | 48.6 TB | 5.1 | 7% |

#### (b) Advertising

| GC Method | GC I/O | Avg Locality | High-locality Ratio |
|-----------|-------:|-------------:|--------------------:|
| BlobDB GC | 266.8 TB | 8.2 | 13% |
| LTGC $\theta=2,\tau=4$ | 163.2 TB | 1.7 | 4% |
| LTGC $\theta=3,\tau=5$ | 87.3 TB | 2.8 | 6% |
| LTGC $\theta=4,\tau=6$ | 58.6 TB | 5.9 | 7% |

**说明**：$(3,5)$ 在 GC I/O 与局部性之间取得最佳综合平衡；更激进参数局部性更好但 I/O 更大，更保守参数反之。

### Table 5：Cost-Efficient Cooperative I/O Engine 消融

| TokaDB Variant | E-commerce P99 | Advertising P99 |
|----------------|---------------:|----------------:|
| **With All Methods** | **71 ms** | **87 ms** |
| - Adaptive Backup Reads | 117 ms | 132 ms |
| - Hardware-Aware Block Alignment | 94 ms | 113 ms |
| - Log as Data | 89 ms | 124 ms |

**说明**：Backup Reads 对慢副本最关键；Block Alignment 验证跨块 seek 的影响；Log as Data 虽直接优化写入，但释放 HDD 带宽后也显著改善读取 P99。

---

## 实验

### 数据集与环境

- **真实负载**：ByteDance E-commerce、Advertising。
- **规模**：每个场景 1,000 万用户、20 个模型；行为数据 24.52 / 37.11 TB，KV Cache 122 / 167 TB。
- **Compute**：6 节点；每节点 Intel 128 logical cores、512 GB RAM、双口 25 GbE。
- **Storage**：24 × Samsung PM1733 7 TB NVMe SSD；72 × Seagate Exos X18 18 TB HDD。
- **基线**：HBase、Doris、Doris+；均运行在 HDFS。TokaDB 同时测试 ByteStore 与 HDFS。

### 端到端结果

- E-commerce：TokaDB on ByteStore 相比 HBase 5×、Doris 3×；总时间 65 min。
- Advertising：相比 HBase 5×、Doris 4×；总时间 77 min。
- TokaDB on HDFS 比 ByteStore 版本慢 31% / 28%，但两场景仍至少领先其他系统 2.9×。
- 摘要给出的总体结果为 **4–10×** 端到端加速；具体倍数随 workload 与比较维度变化。

### 五维性能

1. **Point Read Throughput**：10K 配置下领先 HBase 1.8×、Doris 1.6×、Doris+ 1.3×。
2. **Range Scan Bandwidth**：最高领先 HBase 3.6×、Doris 2.5×、Doris+ 1.7×。
3. **P99 Read Latency**：同吞吐最多降低 70%，100 ms 约束下 QPS 最多提高 30%。
4. **Write Bandwidth**：领先 HBase 52%，领先 Doris / Doris+ 60%。
5. **Cost**：日成本 655.02 USD，较基线获得 2–3× 成本效率。

### 设计验证

- **LTGC**：$(\theta,\tau)=(3,5)$ 为推荐平衡点。
- **Hot-Cold Tiering**：$F_{HDD} \le 0.7$ 基本不损伤读吞吐。
- **I/O Engine**：三项技术都降低 P99，Adaptive Backup Reads 贡献最大。
- **Resource Manager**：抢占调度主要保护尾延迟；缓存隔离和限流同时保护吞吐与延迟。

---

## 批判性思考

### 优点

1. **问题拆分与机制一一对应**：数据布局、I/O、资源隔离分别对应 LRM 的三类根本矛盾，系统结构清晰。
2. **跨层协同而非单点优化**：从逻辑 record / segment 到物理 block、从 user-level scheduler 到设备 token，真正覆盖数据库全栈。
3. **将成本视为一等指标**：不是简单用 SSD 堆性能，而是围绕 HDD 的弱点设计 tail-latency 与写放大优化。
4. **消融充分**：对 LTGC 参数、冷热比例、I/O 三组件、资源管理三组件分别做验证，能解释整体收益来源。
5. **生产真实性强**：真实电商 / 广告负载、百 TB 数据集、明确硬件和公有云价格，工程参考价值高。

### 局限性

1. **核心底座 ByteStore 不公开**：TokaDB on HDFS 虽仍领先，但最强结果依赖内部存储系统，外部难以完全复现。
2. **基线覆盖有限**：只与 HBase / Doris 系列比较；Magnus、Krypton 因不可用未纳入，也缺少更现代 KV-separation 引擎的直接端到端对比。
3. **一致性语义交代不足**：论文强调 durability 与 crash consistency，但多接口、ValueLog 指针更新、冷热迁移期间的事务 / snapshot 语义没有展开。
4. **空间放大数据不足**：Block Alignment 用 padding 换稳定 I/O，但没有单独量化 padding 的真实容量开销分布。
5. **控制参数较多**：$\theta,\tau,\gamma,F_{HDD},S_0,\eta$ 以及多层 token rate 需要调优，尚未形成统一的自动控制框架。
6. **成本模型有利于 TokaDB**：基线无法满足 100 ms 后被放宽到 160 ms；该比较说明基线更弱，但并非严格等 SLA 成本比较。

### 潜在改进方向

1. 用 workload-aware controller 联合自动调节 LTGC、冷热分层、backup deadline 和 rate limit 参数。
2. 为 Hardware-Aware Block Alignment 增加可变大小 packing：在保证单次 I/O 的同时降低小 value padding 浪费。
3. 引入端到端一致性验证与故障注入，量化 ValueLog GC、冷热迁移、log replication 的恢复边界。
4. 面向多租户 / 多模型进一步细化 priority，而不仅是 serving / training 二级划分。
5. 发布 HDFS 版本或 trace-driven simulator，提高学术复现性。

### 可复现性评估

- [ ] 代码开源
- [ ] ByteStore 可获取
- [x] 硬件与主要负载规模给出
- [x] 核心算法参数给出
- [x] 消融实验完整
- [ ] 真实数据集 / trace 可获取

---

## 关联笔记

### 基于

- [[LSM-tree]]：Index SSTable、compaction 与 Log as Data 的索引基础。
- [[SSTable]]：Detached-Segment Index Layout 的紧凑索引载体。
- [[WAL]]：Log as Data 要消除的传统双写路径。
- [[KV Cache]]：TokaDB 低成本协同 I/O 引擎的核心大 value 负载。
- [[ByteStore]]：TokaDB 生产评测的主要底层存储。

### 对比

- [[HBase]]：行存 KV 基线，用户聚合较好但时间过滤和范围扫描较弱。
- Apache Doris：列存基线，时间过滤较好但用户序列拼接有 join 开销。
- Doris+：按用户进一步优化的列式布局，仍无法同时解决所有 LRM 维度。

### 方法相关

- [[Detached-Segment Index Layout]]：用户序列的混合访问数据组织。
- [[Locality-Triggered Garbage Collection]]：按用户碎片度触发局部性恢复。
- [[Hot-Cold Tiering]]：以 segment 为单位做 SSD/HDD 分层。
- [[Adaptive Backup Reads]]：用闭环 deadline 隐藏慢副本。
- [[Hardware-Aware Block Alignment]]：以物理块边界约束大 value 布局。
- [[Log as Data]]：复用日志为主数据，消除双写。
- [[Preemptive Thread Scheduler]]：保护高优先级 serving 的 CPU 调度。
- [[Hierarchical Cache Allocator]]：隔离 metadata、serving、training 缓存。
- [[Multi-Layer Integrated Rate Limiter]]：请求层与硬件层协同限流。

---

## 速查卡片

> [!summary] TokaDB（SIGMOD Companion 2026）
> - **核心**：统一承载 LRM 用户行为序列与 KV Cache 的训练—服务存储引擎。
> - **方法**：Detached-Segment + LTGC / Tiering；Adaptive Backup Read + Block Alignment + Log as Data；抢占调度 + 分层缓存 + 双层限流。
> - **结果**：4–10× 端到端加速，2–3× 成本效率；P99 最多降低 70%，range scan 最高提升 3.6×。
> - **生产**：ByteDance 电商与广告负载，百 TB 级数据，HDD + SSD 混合部署。
> - **代码**：未开源。

---

*笔记创建时间：2026-07-28*
