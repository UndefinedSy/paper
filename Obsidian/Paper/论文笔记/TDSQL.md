---
title: "Reaching the Pinnacle of TPC-DS: Co-design of Architecture, Executor, and Storage in TDSQL"
method_name: "TDSQL"
authors: [Yiteng Chu, Jie Jiang, Yuxing Chen, Feng Zhang, Yuxin Tang, Anqun Pan, Xin Wu, Xiang Hu, Hailin Lei, Qian Zhang, Ziyan Lin, Xiaoliang Yan, Xiaojian Fan, Liang Zhang, Bin Qu, Wen Zhang, Mengbang Xing, Lixiong Zheng, Wei Lu, Yunpeng Chai, Xiaoyong Du]
year: 2026
venue: PVLDB 19(12)
tags: [olap-database, mpp-architecture, vectorized-execution, columnar-storage, runtime-filter, tencent, tpc-ds]
zotero_collection: VLDB/2026
image_source: local
created: 2026-09-24
---

# 论文笔记：Reaching the Pinnacle of TPC-DS: Co-design of Architecture, Executor, and Storage in TDSQL

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | Renmin University of China、Tencent Inc. |
| 日期 | 2026，PVLDB 19(12): 4009–4022 |
| 项目主页 | — |
| 对比基线 | [[Greenplum]]、[[PrestoDB]]、[[DuckDB]]、TeleDB、AnalyticDB、[[Snowflake]]、Databricks |
| 链接 | [DOI](https://doi.org/10.14778/3827998.3828012) |
| 原始 PDF | `VLDB/2026/Co-design of Architecture, Executor, and Storage in TDSQL.pdf` |

---

## 一句话总结

> TDSQL 通过 Forward Node 通信层、Virtual DN 算子级并行、全链路向量化执行引擎与分组级联自适应 Runtime Filter 的系统性协同设计，在官方 TPC-DS 10,000 GB 测试中拿下全球第一，QphDS 达 7260 万，是第二名的 1.81 倍。

---

## 核心贡献

1. **Forward Node 架构**：在传统 MPP 拓扑上引入数据转发中间层 Forward Node，把物理全连接（$O(N^2)$）拓扑收敛为跨机 $O(n(n-1))$ 的星型拓扑，缓解"连接爆炸"问题。
2. **协同设计的并行化与向量化**：Fragment 级引入 push + Pipeline 执行模型，算子级提出 [[Virtual DN Parallelism|Virtual DN 并行]] 策略，配合 non-intrusive 向量化执行引擎与 Estore 列存引擎，系统性提升 CPU 利用率。
3. **分层自适应 Runtime Filter**：提出 group-wise cascading 通信策略降低广播开销，并基于统计信息与历史滑动窗口设计自适应构建/绕过策略，避免低选择率场景下的性能倒退。
4. **官方 TPC-DS 全球第一**：10,000 GB 配置下 QphDS 达 72,603,042，领先第二/第三名 1.81×/3.82×，单位性能成本降低 79%/37%。

---

## 问题背景

### 要解决的问题

企业数据爆炸式增长与复杂多维实时分析需求，要求 OLAP 数据库同时具备高可扩展性、高效率与高性价比。TDSQL 是腾讯自 2008 年孵化、2019 年开源的企业级分布式数据库，服务超 500,000 用户、4,000+ 企业客户（含中国七大银行），需支撑每月 300+ TB 的支付/电信数据实时接入与查询。

### 现有方法的局限

论文用 Table 1（见"关键图表"）对比四类开源代表系统（[[Greenplum]]、[[PrestoDB]]、[[ClickHouse]]、[[DuckDB]]）后指出三大架构级缺陷：

1. **传统 [[MPP Architecture|MPP]] 架构连接爆炸**：$n$ 台服务器、$N$ 个 Datanode 场景下，数据重分布阶段需要 $O(N^2)$ 点对点连接；若并发 $X$ 个查询、每个查询涉及 $M$ 层 Join，连接规模进一步膨胀为 $O(X\cdot M\cdot N^2)$，消耗内存并带来 socket 创建/维护开销。云原生解耦架构（Databricks、BigQuery）用外部 shuffle 服务规避连接爆炸，但引入磁盘 I/O 与多跳网络开销。
2. **CPU 利用率低**：主流系统缺乏完整算子级并行与原生向量化执行；直接把 [[DuckDB]] 式范式硬套进成熟架构，会带来严重的节点内协调开销与遗留兼容性障碍。
3. **[[Runtime Filter]] 静态脆弱**：传统全连接广播式 runtime filter 在低选择率场景下，构造与传输的固定开销可能超过其剪枝收益，反而拖慢查询（[[ClickHouse]] 式纯执行期绕过仍要承担初始构建/传输的沉没成本）。

### 本文的动机

TDSQL 保留原生 MPP 低延迟数据搬移的优势，通过系统性协同设计（communication、parallelization/vectorization、runtime filter 三条主线）逐一击穿上述瓶颈，而非推翻现有成熟生态重做架构。

---

## 方法详解

### 系统架构总览

Figure 2 给出 TDSQL 的核心架构：[[Global Transaction Manager|GTM]] 管理全局事务，Coordinator 负责查询分发，[[Forward Node]] 层承担节点间数据转发，Datanode 管理存储与执行；存储层由行存与列存引擎 [[Estore]]（Silos + Stash + Registry）组成；计算层包含并行执行引擎（Push Model + Pipeline Model + [[Virtual DN Parallelism|Virtual DN 并行]]/随机并行）、向量化执行引擎与自适应 Runtime Filter 模块。

### Figure 2: 系统整体架构

![[assets/TDSQL/TDSQL_fig2_architecture.png]]

**说明**：GTM/Coordinator/Datanode 组成的 MPP 主体之上叠加 Forward Node 通信层；每个 Datanode 内部由存储层（Row Storage + Estore 列存）与计算层（并行执行引擎、向量化执行引擎、自适应 Runtime Filter）构成。四项创新（Forward Node、[[Virtual DN Parallelism]]、[[Vectorized Execution Engine|向量化执行引擎]]、[[Adaptive Runtime Filter|自适应 Runtime Filter]]）分别对应 Section 4/5/6/7。

### 模块 1：Forward Node 通信层

#### 设计动机

消除传统全连接（full-mesh P2P）拓扑随集群规模呈平方级增长的连接开销，同时避免退化为高延迟外部 shuffle 存储服务。

#### 具体实现

TDSQL 采用多 Datanode、多进程部署模型做进程级隔离；引入 Forward Node（FN）作为每台服务器的数据通信代理：

- **服务器内通信**：Coordinator/Datanode 与本地 FN 通过 [[Shared Memory|共享内存]] 零拷贝交互，无需建立连接。
- **跨服务器通信**：所有跨机数据经由本地 FN 路由，FN 之间做连接复用；集群连接数从 $O(N^2)$ 降为 $O(n\cdot(n-1))$（$n$ 为服务器数）。
- **模块化设计**（Figure 3）：元数据管理模块维护 FN 与 Coordinator/Datanode 的进程标识等通信元数据；分布式共享内存模块为每个 Datanode 分配专用内存区；Sender/Receiver 模块通过共享内存 + 消息队列 + 调度线程实现双向异步数据传输。

### Figure 3: Forward Node 传输架构

![[assets/TDSQL/TDSQL_fig3_forward_node.png]]

**说明**：Client 的查询/结果经 Coordinator 与各 Server 上的 Forward Node 交互；放大视图展示单个 FN 内部的 Dispatch Thread、Meta Data、Sender（Fragment Message Queue + Merge Thread + Sender Message Queue + Sender Thread + Sender Shared Memory）与 Receiver（对称结构）。

### Figure 4: 跨 FN 数据通信细节

![[assets/TDSQL/TDSQL_fig4_fn_communication.png]]

**说明**：Server 1 的 DProcess 写入 Fragment Message Queue → Sender FN 通过 Sender Message Queue 与 Sender Buffer Pool 完成网络发送；Server 2 的 Receiver FN 接收后写入 Receiver Buffer Pool 与 Receiver Message Queue，再分发回目标 DProcess，实现发送与接收解耦的异步流水线。

### 模块 2：并行执行框架

#### Fragment 级：Push + Pipeline

不同于 [[Greenplum]]/[[Impala]] 的拉取式（Volcano 迭代器模型）执行，TDSQL 采用推送模型：下层 Fragment 主动把数据推给上层 Fragment，并引入 data thread 做异步预取与数据结构转换，消除按需驱动带来的同步等待。为防止资源受限环境下的过度并行造成系统过载，TDSQL 进一步引入 Pipeline 机制：按 Fragment 间的生产者-消费者关系组织成多个 Pipeline，在保证单 Pipeline 并行度不超过系统最大并行度的前提下动态分配并行线程。

### Figure 5: 并行查询计划示例

![[assets/TDSQL/TDSQL_fig5_parallel_plan.png]]

**说明**：一条四表 Join + GroupBy + Having 的查询被拆分为 5 个 Fragment；Fragment 间通过 push 模型异步执行，RemoteSubplan 算子负责跨节点数据重分布，虚拟 Datanode（VDN1/VDN2）内各自完成 Hash + Hashjoin，保证分区独立。

### Figure 6: Pipeline 执行方案

![[assets/TDSQL/TDSQL_fig6_pipeline.png]]

**说明**：5 个 Fragment 依据依赖关系组织成 4 个 Pipeline；在最大并行度为 8 的约束下，各 Pipeline 内的 Fragment 按需分配并行线程数（如图中数字 1/2/3/4/5 代表对应 Fragment 的并行线程编号）。

#### 算子级：Virtual DN Parallelism

传统随机并行（tuple 随机分配给 worker、无 key-range 隔离）在窗口聚合、full join 等需要跨 worker 一致分区的算子上容易冲突，且协调/数据交换开销大。TDSQL 提出 [[Virtual DN Parallelism|Virtual DN 并行]]：在节点内部模拟多个虚拟 Datanode，每个 VDN 拥有独立数据范围并自治执行算子；[[RemoteSubplan]] 算子作为节点内分布边界，按下游分区需求重分布 tuple，保证跨 VDN 数据独立。为兼顾适用范围，TDSQL 采用**混合并行策略**：仅当算子上游存在提供 key 分区输入的 RemoteSubplan 时才启用 Virtual DN Parallelism，否则回退到随机并行；RemoteSubplan 还基于数据分布统计做 range-based 重分区以缓解数据倾斜。

### Figure 7: Estore 存储设计

![[assets/TDSQL/TDSQL_fig7_estore_storage.png]]

**说明**：每列数据按 Silo 分片存储（固定行数、可变物理大小）；Registry 表作为元数据目录，记录每个 Silo 的 `col_id`、`silo_id`、`silo_prt`、`row_count`、`min/max` 等；Stash 表以行存形式临时缓冲小批量/流式/OLTP 混合写入，达到阈值后周期性 merge 回 Silo，避免 Registry/Silo 布局碎片化。

### 模块 3：系统化向量化设计

#### 设计动机

在不破坏遗留标量执行路径兼容性的前提下，把向量化/列存能力非侵入式集成进 TDSQL 成熟生态。

#### 具体实现

1. **向量化执行框架**：自底向上贪心遍历并重写计划树，若子树支持向量化则整体以向量模式执行，否则在向量/非向量算子间插入 row-vector 转换适配器，兼容遗留执行引擎。
2. **内存数据结构**（Figure 8）：引入 [[VectorTableSlot]] 承载多个 [[ColumnVector]]；`ColumnVector` 对 ≤8 字节定长数据直接存入 `vals` 数组，超长/变长数据经 `buf` 指针间接存储，`flags` 数组维护 null 状态与压缩标记；[[VectorBuffer]] 以 `TinyBuffer` 链表形式统一管理内存块，减少跨算子内存拷贝与缓存缺失。
3. **算子重构**：对 [[RemoteSubplan]]、Scan、[[HashAgg]]、[[HashJoin]] 等关键算子做批处理化重构：
   - HashAgg 采用 xxHash + 线性探测的两级 Pointer Block–Payload Block 哈希表；超出内存阈值时溢出到磁盘；SelectionVector 把批量探测结果划分为 absent-key/existing-key/collision 三组分别处理，减少分支并配合 SIMD 加速哈希与键比较。
   - HashJoin 的 build/probe 阶段均批处理化，probe 阶段匹配位置记录进 bitmap 实现无分支批量输出，SIMD 加速哈希/比较，构建与探测均可任务级并行。
4. **向量化标量函数**：提出 metadata-driven 的标量-向量函数映射机制，执行期用二分查找定位并调用向量化实现，避免重写数千个既有函数。

### Figure 8: 向量化内存数据结构

![[assets/TDSQL/TDSQL_fig8_memory_structures.png]]

**说明**：`VectorTableSlot` 持有元数据（rows/cols）与多个 `ColumnVector`；每个 `ColumnVector` 含 Header（flags/vals/bufs）；`VectorBuffer` 以多个 `TinyBuffer` 链表节点管理连续可用内存块，减少跨算子内存拷贝。

### 模块 4：Runtime Filter 优化

TDSQL 实现三类 Runtime Filter：[[Bloom Filter]]、Range Filter、In Filter。

#### Group-wise Cascading 通信策略

同一物理机内的多个 Datanode 组成一个 group，由该机的 Forward Node 聚合各 Datanode 本地生成的 filter 形成 group-level filter；随后每个 group 的 FN 只做一次跨 group 广播；目标机 FN 收到外部 group-level filter 后与本地 filter 合并成全局 filter 再分发给本机所有 Datanode，将全连接"广播爆炸"转化为受控的分层交换。

#### 统计信息驱动的自适应策略

给定表 $T$（probe 端）与 $S$（build 端）在属性 $Z$ 上的 Join，构建阶段先用 NDV（Number of Distinct Values）估计过滤有效性：

### 公式1：[[Runtime Filter|Filter 过滤有效性估计]]

$$
\text{filtering\_effectiveness} = 1 - \frac{NDV_S(Z)}{NDV_T(Z)}
$$

**含义**：估计构建一个 runtime filter 对 probe 端的预期剪枝效果；若低于预设阈值 $\beta$ 则直接禁用该 filter 的构建。

**符号说明**：
- $NDV_T(Z)$：probe 端表 $T$ 在属性 $Z$ 上的不同值个数；
- $NDV_S(Z)$：build 端表 $S$ 在属性 $Z$ 上的不同值个数；
- $\beta$：系统预定义的启用阈值。

启用后，在 probe 阶段引入 history-aware 的滑动窗口机制持续监控前序批次过滤率，动态决定当前批次是否继续应用 filter（见下方算法 1）：

### 算法 1：动态自适应过滤算法

![[assets/TDSQL/TDSQL_alg1_adaptive_filter.png]]

**含义**：对批次序列 $X=\{X_1,\dots,X_n\}$，令 $F_i = |x_i|/|X_i|$ 为第 $i$ 批的过滤率（$|x_i|$ 为该批被过滤掉的记录数）。若第一批或 $F_{i-1}\ge\alpha$，则启用 filter；若仅上一批过滤率不足但上上批（或第二批）达标，则本批禁用；若连续两批过滤率均不足 $\alpha$，则对当前及下一批同时禁用 filter。该机制避免在低选择率批次上白白承担 filter 应用开销。

---

## 关键图表

### Figure 1: TPC-DS 公开报告延迟对比

![[assets/TDSQL/TDSQL_fig1_tpcds_latency.png]]

**说明**：SuperMicro/H3C/E-MR/TeleDB/ADB/TDSQL 六个系统在 TPC-DS 公开报告中的 99 查询总延迟，TDSQL 以 732 秒远低于其余系统（ADB 2509s、TeleDB 2812s）。

### Table 1: 开源分析型数据库特征对比

![[assets/TDSQL/TDSQL_table1_comparison.png]]

**说明**：对比 [[Greenplum]]、[[PrestoDB]]、[[ClickHouse]]、[[DuckDB]]、TDSQL 在 SQL 语法支持、MPP 架构、集中式通信、完整算子并行、向量化执行引擎、自适应 Runtime Filter、列存等 7 个维度的支持情况；TDSQL 是唯一同时具备"集中式通信"与"自适应 Runtime Filter"的系统。

### Table 2: TPC-DS 官方公开报告排名

![[assets/TDSQL/TDSQL_table2_tpcds_report.png]]

| Rank | System | Performance (QphDS) | pt_time | tt_time | Price/kQphDS (CNY) |
|---|---|---|---|---|---|
| 1 | TDSQL | 72.60M | 372.9s | 731.9s | ¥37.52 |
| 2 | TeleDB | 40.20M | 847.8s | 2811.6s | ¥179.73 |
| 3 | AnalyticDB | 18.99M | 849.9s | 2508.5s | ¥59.27 |

**说明**：10,000 GB 配置下，TDSQL 以 128 节点集群拿到全球第一，QphDS 分别领先 TeleDB、AnalyticDB 1.81×、3.82×；单位性能成本比二三名低 79%/37%。数据加载 10,000 GB 仅需 319.1 秒。

### Figure 9: 与开源/云系统对比（1000 GB）

![[assets/TDSQL/TDSQL_fig9_opensource_cloud.png]]

**说明**：PrestoDB/GreenPlum/Databricks/DuckDB/TDSQL-1（单节点）/TDSQL 六者 pt_time 对比；单机版 TDSQL-1 达 250.0s，优于单节点 [[DuckDB]] 的 539.3s；分布式 TDSQL 达 179.4s 为最优。

### Table 3: 10 TB TPC-DS 云仓库对比

![[assets/TDSQL/TDSQL_table3_cloud_warehouse.png]]

| System | TDSQL | Databricks | Snowflake |
|---|---|---|---|
| pt_time (s) | 372.90 | 782.36 | 726.15 |

**说明**：在 10 TB 规模下，TDSQL 单流耗时比 Databricks（X-Large/5X-Large）、Snowflake（6X-Large）分别快 2.10×、1.95×（云仓库规格非硬件对齐，仅作实用基线参考）。

### Figure 10: 消融实验总览

![[assets/TDSQL/TDSQL_fig10_ablation_variants.png]]

**说明**：GreenPlum 与 TDSQL 全量/各消融变体（w/o parallelism、w/o vectorization、w/o runtime filter 及其组合）在单流/多流下的总响应时间对比；[[Vectorized Execution Engine|向量化执行引擎]]贡献最大单项收益（降低 70%+ 总耗时），Runtime Filter 降低 65.1%/55.2%（单流/多流），所有 TDSQL 变体均优于 GreenPlum。

### Figure 11: 消融对 99 查询延迟的影响（Top-10）

![[assets/TDSQL/TDSQL_fig11_ablation_queries.png]]

**说明**：三张子图分别展示并行优化（提升 83% 查询，53% 提速 >50%）、向量化执行（提升 78% 查询，46% 提速 >50%）、Runtime Filter（提升 83% 查询，29% 提速 >50%）各自加速最显著的 10 条查询；并行优化对多级 GroupBy/Window Agg 收益最大，向量化对计算密集型聚合/排序收益最大，Runtime Filter 对大表 Join 收益最大。

### Figure 12: 并行优化的 CPU 利用率

![[assets/TDSQL/TDSQL_fig12_cpu_parallel.png]]

**说明**：单流/多流场景下，启用并行优化后平均 CPU 利用率分别提升 35%/6%，且各节点利用率一致，说明负载均衡良好。

### Figure 13: 向量化执行的 CPU 利用率

![[assets/TDSQL/TDSQL_fig13_cpu_vectorization.png]]

**说明**：未启用向量化时 CPU 利用率因大量 I/O 等待而剧烈波动；启用后单流/多流平均 CPU 利用率分别提升 24%/12%，全程保持高位。

### Figure 14: 可扩展性评估

![[assets/TDSQL/TDSQL_fig14_scalability.png]]

**说明**：(a) 1TB→16TB 数据规模下，TDSQL 延迟增长平缓（单流 179.38s→437.19s），[[Greenplum]] 急剧恶化（778.99s→8377.55s），16TB 规模下差距达约 19×；(b) 16→128 节点，TDSQL 延迟随节点数增加持续下降，得益于 Forward Node 架构在大规模集群下降低连接管理开销。

### Table 4: 真实业务场景对比（TDSQL vs 匿名对手 OP）

![[assets/TDSQL/TDSQL_table4_real_workload.png]]

| Database | W1 | W2 | W3 | W4 | W5 | W6 |
|---|---|---|---|---|---|---|
| OP | 1292ms | 93ms | 433ms | 545ms | 53ms | 61ms |
| TDSQL | 423ms | 39ms | 195ms | 287ms | 29ms | 34ms |

**说明**：PSI（安防，覆盖数十亿移动设备）与 Retail（零售全渠道运营分析）两大真实场景的 6 个代表性查询任务（大表扫描 W2/W5、复杂 Join W3/W6/W1、聚合密集 W4），TDSQL 在所有场景至少领先 40%。TDSQL 已部署超 15,000 实例，从 PostgreSQL 等系统迁移的整体性能提升超 50%。

---

## 实验

### 实验环境

- **集群**：8 台物理机，每台 Intel Xeon Platinum 8163 @2.50GHz（112 vCPU）、1024 GB 内存；两层存储（2×960GB SATA SSD 系统盘 + 16×7.68TB NVMe SSD 数据盘）；100 Gbps 银杉智能网卡互联。
- **对比对象**：开源 [[Greenplum]]、[[PrestoDB]]（Apache ORC + HDFS）、[[DuckDB]]（单机向量化，无集群能力）；商业系统 AnalyticDB、TeleDB（经官方 TPC 报告对比）；云仓库 Databricks（X-Large / 5X-Large）、[[Snowflake]]（6X-Large），均运行于 Microsoft Azure。
- **Workload**：官方 TPC-DS，默认 scale factor 10000（10 TB）；单流/多流两种并发设置；另评估真实生产 OLAP 场景（安防分析、零售分析）。

### 主要结果

- 10,000 GB 官方 TPC-DS：QphDS 72,603,042，全球第一；单流/多流总耗时 372.9s/731.9s；数据加载 319.1s；成本较第二/第三名低 79%/37%。
- 1000 GB 开源/云对比：TDSQL 179.4s，[[DuckDB]] 539.3s（单机），单机版 TDSQL-1 为 250.0s。
- 4.34× 高于 [[Greenplum]] 的查询性能与更优扩展性；单节点场景下 2.16× 低于 [[DuckDB]] 的延迟；真实业务场景至少领先对手 40%。

### 讨论与洞见

- **CPU 瓶颈而非内存瓶颈**：与 TPC-C 的内存瓶颈不同，TPC-DS 复杂查询并行执行时 CPU 利用率超 90%，内存仅 60%，表明进一步 CPU 优化（新硬件、GPU 卸载、专用分析处理器）仍有较大提升空间。
- **并行度并非越高越好**：过高并行度会导致明显性能下降，当前依赖经验设定最大并行度；未来计划结合实时负载分析和 AI 辅助决策动态调整并行度。

---

## 批判性思考

### 优点

1. **务实的非侵入式协同设计**：不追求推翻成熟 MPP 架构重做，而是在 Forward Node、向量化、Runtime Filter 三个维度做增量式协同设计，兼顾工程可落地性与性能收益，符合大型商业数据库长期演进的现实约束。
2. **消融实验详尽**：Figure 10/11/12/13 从整体到单查询、从延迟到 CPU 利用率多角度验证了每个优化组件的独立贡献与叠加效应，说服力较强。
3. **真实场景验证**：不仅有 TPC-DS 官方成绩，还给出 15,000+ 实例部署规模与安防/零售真实业务的对比数据，弥补了纯 benchmark 论文常见的"实验室数据"局限。

### 局限性

1. **对比商业系统信息不透明**：TeleDB、AnalyticDB 的内部实现细节未知，仅通过官方 TPC 报告间接对比，无法做细粒度归因分析。
2. **自适应 Runtime Filter 的阈值 $\alpha$、$\beta$ 依赖经验设定**，论文未给出具体取值范围或自动调参方法，可复现性和跨工作负载泛化能力存疑。
3. **并行度调优仍依赖人工经验**，论文自己也承认这是未来工作方向，当前方案在客户自助场景中可能过于复杂。

### 潜在改进方向

1. 论文提出的"AI 辅助实时并行度调整"值得后续跟进，可参考同类的 learned data partitioning 方法（如 LRP，Liu et al. 2025，见参考文献 [42]）。
2. Runtime Filter 的统计驱动阈值可考虑用在线学习方式自动校准，减少人工经验依赖。
3. CPU 利用率瓶颈提示了 GPU 加速 Join/Agg 的潜在收益方向（论文也在 Discussion 中提及）。

### 可复现性评估
- [ ] 代码开源（TDSQL 底层 TBase 开源，但本文的 Forward Node/向量化/Runtime Filter 具体实现细节未随论文公开代码）
- [ ] 预训练模型（不适用）
- [ ] 训练细节完整（不适用，非 ML 论文）
- [x] 数据集可获取（TPC-DS 标准 benchmark 公开可获取）

---

## 关联笔记

### 基于
- TDSQL: tencent distributed database system（VLDB 2024，OLTP 前作）：本文聚焦 OLAP 优化，是同一系统在分析场景的延伸工作。

### 对比
- [[Greenplum]]：传统 MPP 架构代表，全文多处作为主要对比基线。
- [[ClickHouse]]：Push 执行模型的先行者，也是 Runtime Filter 静态绕过机制的对比对象。
- [[DuckDB]]：单机向量化执行代表，被用于验证 TDSQL 单机模式（TDSQL-1）的计算效率。
- [[Snowflake]]：云原生解耦仓库代表，用于 10TB 场景下的实用云基线对比。

### 方法相关
- [[Runtime Filter]]：本文的分层自适应机制是对该概念的重要扩展。
- [[Bloom Filter]]：TDSQL 实现的三类 Runtime Filter 之一。
- [[LSM-tree]]：Estore 混合行列存储借鉴了 LSM 式的 Stash（写缓冲）+ Silo（列存）合并思想。
- [[SIMD]]：向量化 HashAgg/HashJoin 与标量函数加速的底层硬件基础。

### 硬件/数据相关
- [[TPC-DS]]：本文的核心评测 benchmark。

---

## 速查卡片

> [!summary] Reaching the Pinnacle of TPC-DS: Co-design of Architecture, Executor, and Storage in TDSQL
> - **核心**：Forward Node 通信 + Virtual DN 并行 + 全链路向量化 + 分组级联自适应 Runtime Filter 的系统性协同设计
> - **方法**：MPP 架构上叠加数据转发中间层、push/pipeline 执行模型、非侵入式向量化引擎（Estore 列存）、统计驱动的自适应 Runtime Filter
> - **结果**：官方 TPC-DS 10,000GB 全球第一，QphDS 7260万，领先第二/三名 1.81×/3.82×，成本降低 79%/37%
> - **代码**：TDSQL/TBase 部分开源（https://github.com/Tencent/TBase），本文优化细节未随文开源

---

*笔记创建时间: 2026-09-24*
