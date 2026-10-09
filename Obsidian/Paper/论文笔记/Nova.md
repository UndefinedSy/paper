---
title: "Nova: A Multi-Purpose Vector Engine for Low-Latency, Multi-Tenant, and Cross-Table Hybrid Retrieval"
method_name: "Nova"
authors: [Dechuang Chen, Bing Chen, Jianwei Lu, Zhiwei Wu, Zeyu Yang, Qiang Gu, Fangyuan Zhang, Sibo Wang, Zeyuan Yu, Caihua Yin, Wenchao Zhou, Feifei Li]
year: 2026
venue: PVLDB
tags: [vector-database, hybrid-retrieval, multi-tenancy, rag, ann-search, delta-base, wal-durability]
zotero_collection: VLDB/2026
image_source: local
created: 2026-10-08
---

# 论文笔记：Nova

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | 阿里巴巴云（AnalyticDB-PG 团队）；香港中文大学；哈尔滨工业大学（深圳） |
| 发表 | PVLDB 19(12), 2026, pp. 4453–4465（VLDB 2026） |
| DOI | [10.14778/3827998.3828045](https://doi.org/10.14778/3827998.3828045) |
| 对比基线 | [[Milvus]]、[[PGVector]]、[[Elasticsearch]]、[[ADB-V]]、PostgreSQL-V |
| 适用场景 | [[RAG]] 知识库检索、多租户向量服务、结构化—非结构化跨表混合检索 |
| 落地规模 | 40 万+ 用户、60 万+ 数据库、1000 亿+ 向量条目 |

---

## 一句话总结

> Nova 用 delta-base 双层架构 + 文件式 mmap 存储，把向量检索变成关系优化器里的一等公民，实现低延迟写入、多租户隔离与跨表混合检索。

---

## 核心贡献

1. **Delta-Base 双层架构**：把实时写入与基索引解耦，通过 [[Delta File]] 环形队列实现低延迟写入、写后立即可见与基索引无锁扫描；配套两级 [[WAL]] 持久化与多租户优化（Sec. 3）。
2. **两个场景化索引**：[[Nova Memory]]（基于 [[HNSW]] 的图索引，带重复向量合并与空间复用）与 [[Nova Disk]]（[[Inverted File Index|IVF]] 混合索引，带自适应扫描与动态扩容），覆盖内存驻留与磁盘驻留两类业务（Sec. 4）。
3. **首个跨表混合检索的系统性处理**：把 [[Partial Index]] / [[Covering Index]] 与 [[Cost-Based Adaptive Join|基于代价的自适应连接]] 整合进扩展后的优化器，打通向量扫描与关系连接的边界（Sec. 5）。
4. **生产级稳定性验证**：公开数据集（[[GIST Dataset|GIST]]、[[Cohere Dataset|Cohere]]）与内部数据集（[[Tongyi Dataset|Tongyi]]）上全面优于 Milvus、pgvector 等系统，在 [[Alibaba Cloud Model Studio|Model Studio]] 完成 24 小时真实压测（Sec. 6）。

---

## 问题背景

### 要解决的问题

向量检索已成为推荐、图像检索与 [[RAG]] 的底层支柱。生产环境对向量引擎提出四类并发要求，而现有系统难以同时满足：

1. **资源争抢下的低延迟与鲁棒性**：向量负载天然具备突发写特征——前端一次文档上传可能被切成数万个 chunk，在短时间内批量嵌入并插入；同时 `vacuum` 等后台进程会周期性争抢 CPU 与 I/O。数据库必须在扰动下维持稳定的查询与写入延迟。
2. **高密度多租户**：[[Alibaba Cloud Model Studio|Model Studio]] 单个实例要服务 1 万–3 万用户、平均 300 并发请求。为每个用户单独建实例在资源与运维成本上不可行，共享实例又必须防止 [[Noisy Neighbor|"吵闹邻居"]] 效应，要求索引结构的存储与计算开销都足够轻量，且资源可跨租户动态调度。
3. **跨表混合查询**：单表混合查询（向量检索 + 标量过滤）已有大量研究，但实践中结构化数据与非结构化数据常分表存放。RAG 场景下 chunk 向量存于 `t_embedding`，文件名、标签等元数据存于 `t_metadata`，两表以 `doc_id` 关联。
4. **One Size Does Not Fit All**：生化企业用向量表示分子结构，库规模达数百亿条，内部使用、并发低但极度成本敏感；社交网络用向量刻画用户行为做实时匹配，则要求极低查询延迟与高稳定性。单一方案无法同时覆盖。

### 现有方法的局限

- **合并成大宽表的代价**：把元数据与向量合并到一张宽表会导致元数据冗余存储，且每行存多个向量使更新昂贵。在 [[MVCC]] 下问题被放大——即使只是元数据变更，也会迫使整条向量记录被复制并重新插入向量索引，造成严重的写放大。
- **页式存储不匹配向量数据**：传统 page-based 布局存在内部碎片。pgvector 使用 8KB 页，而一个 1024 维向量以 4 字节浮点表示就已超过 4KB，一页只能存一个向量加 `ctid` 与元数据，空间浪费严重。其次，页级读写锁会在 `vacuum` 等操作期间拖慢索引扫描。
- **衍生自 ADB-V 的限制**：Nova 的前身 [[ADB-V]] 使用共享缓冲池 + 页级锁，向图的插入直接在页锁下进行，在大批量写入触发 `vacuum` 时会出现查询与 upsert 的 P99 延迟尖峰。

### 本文的动机

作者并不试图发明新的距离度量或 ANN 图算法，而是把已有的 [[HNSW]] / [[Inverted File Index|IVF]] 改造成**可维护的数据库物理结构**：图节点布局要匹配页与维度，增删改要与事务和可见性协作，过滤要落成联合索引，跨表检索要由优化器统一决策。系统整体构建在 AnalyticDB-PG（[[AnalyticDB-PG|ADB-PG]]）之上，沿用其 [[MPP Architecture|MPP]] 架构：一个 coordinator 节点（Parser / Optimizer / Query Dispatcher / DTM）+ 多个 worker 节点，表按行分区分布。

---

## 方法详解

### 总体架构

Nova 在 [[AnalyticDB-PG|ADB-PG]] 之上构建，对外作为 [[Alibaba Cloud Model Studio|Model Studio]] 知识库管理的存储后端。核心是 [[Delta-Base Architecture|delta-base 双层架构]] 与 [[File-Based Storage|基于文件的存储管理]]，并在此框架上实现两个索引变体（[[Nova Memory]] 与 [[Nova Disk]]）。

### 模块 1：Delta-Base 双层框架

**设计动机**：让实时写入路径与索引维护路径彻底分离，从而同时获得低写入延迟、数据立即可见与基索引的无锁查询。

**三大组件**（Figure 3）：

- **[[Delta File|Delta 文件]]**：承接实时写流量，临时存放增量数据；
- **Base Index（基索引）**：底层索引结构，具体形态随索引类型变化（IVF 时为 metadata file + posting list 等结构文件）；
- **[[WAL]]**：故障恢复用，保障索引高可用。

后台进程持续监控 delta 文件状态，当累积向量数超过阈值**或**经过特定时间间隔时触发 `flush`，把增量数据合并进基索引。

**Delta 文件的多重角色**：

1. **环形队列组织**：每个 delta 文件内部是一个 [[Ring Queue|环形队列]]，使插入操作轻量、写入延迟低，并能吸收突发写流量，从而支撑系统稳定性。
2. **双路径召回（two-path recall）**：查询同时探测 delta 文件与基索引。数据写入 delta 文件后**立即可见**，保证实时数据新鲜度。
3. **冲突收敛**：delta 文件把写—写冲突（并发 insert 之间、以及 insert 与 vacuum 之间）**限制在文件内部**，因此基索引上的扫描可以是无锁的——基索引只需管理 `flush`/`vacuum` 与扫描之间的读写并发。
4. **多文件并发**：delta 文件本身用读写锁做并发控制，但操作轻量、持锁时间短；Nova 默认维护**三个** delta 文件以进一步提升并发度。从 delta 文件读取数据有一定开销，但相比基索引无锁执行带来的收益可以忽略。

**文件式存储**：所有数据按原生内存布局直接写入文件，打开索引时通过 `mmap` 直接映射进进程地址空间，使数据可如同内存常驻般操作。`mmap` 机制自动在内存中缓存热数据、在内存紧张时换出冷数据，对上层实现完全透明。该设计假设 POSIX 兼容文件系统，当前部署在本地 NVMe 盘与阿里云 ESSD 云盘上；向对象存储（如阿里云 OSS）的扩展留作未来工作。

**量化支持**：支持 [[Product Quantization|PQ]]、[[RaBitQ]] 等多种量化方案。启用量化时，基索引只存量化后的向量，**delta 文件保留原始向量**，并额外维护 PQ codebook、RaBitQ 旋转矩阵等辅助文件以加速量化向量的距离计算。

**[[Lazy Index Construction|惰性索引构建]]**：部分向量索引需要足够的训练数据（如聚类类方法需要足够向量才能可靠聚类）。Nova 允许在空表上预先创建此类索引并把构建参数持久化到元数据；基索引保持为空，delta 文件在达到采样阈值前不建索引，阈值到达后再触发构建并把缓冲数据灌入基索引。

#### 关键操作

| 操作 | 流程 | 并发语义 |
|---|---|---|
| `scan` | 对 delta 文件与基索引做双路径召回，合并去重；支持 bitmap / predicate 过滤；可用用户可配的**召回放大因子**提升 recall | 对 delta 文件取读锁；基索引**无锁**访问 |
| `insert` | 先写 [[WAL]]，再取写锁把向量与其 `ctid` 追加到某个可用 delta 文件；两者都完成才提交 | delta 文件内受限 |
| `flush` | 先取 delta 文件读锁安全转移条目，再升级为写锁仅做截断；基索引插入分批进行、可选无锁 | 写锁只覆盖极短的截断步骤；转移期间的并发 insert 可路由到其他 delta 文件 |
| `vacuum` | 若基索引存在：先 flush delta，再 vacuum 基索引（也可无锁）；若基索引未建：取 delta 写锁就地进行死元组过滤与压实 | 保证 `flush` 与 `vacuum` **永不并发**，任一刻最多一个写者修改基索引 |

#### 两级 WAL 与持久化顺序

**难点**：WAL 通常要求「日志落盘先于数据落盘」，否则崩溃恢复可能看到不一致。但在 `mmap` 存储下，脏页回写由内核控制、本质上是非确定的——对 mmap 内存的直接写入可能先于其 WAL 记录落盘；而每次写前强制刷 WAL 又会产生大量小 I/O，严重拖慢性能。

**解法**：把 WAL 记录逻辑上分为两类，共享同一条 PostgreSQL WAL 流，但各自有独立的刷盘策略：

- **meta 类**：元数据文件（Figure 3 中的 metadata file）的相关记录。记录小，通过最大 offset、slot 数等边界决定数据可见性；越过边界的数据在扫描时被忽略、由 vacuum 回收。
- **data 类**：结构文件的相关记录。

**顺序保证**：Nova **总是先刷 data WAL 再刷 meta WAL**，事务只有在 meta WAL 落盘后才算持久化。

由于元数据有一致性强要求，Nova 对元数据做了更强的顺序控制：**打开索引时把元数据文件内容复制到一块独立内存位置**，后续所有写入与 vacuum 都在这份内存副本上进行（而非 mmap 文件）。事务提交时先刷 meta WAL，再把元数据变更写回元数据文件。相反，data 类修改直接作用在 mmap 文件上，可由内核异步、乱序落盘。该设计把昂贵的同步 I/O 限制在通常很小的元数据范围内，避免在大体量数据上承担同步写代价，从而在一致性与高吞吐之间取得有利折中。

**检查点**：需要内存状态与磁盘同步时，Nova 通过对索引文件调用 `fsync` 把脏页从 page cache 刷出。

**实现细节**：Nova 通过 Resource Manager 扩展接口注册自定义资源管理器 `RM_NOVA_ID`，把所有索引状态变更记入 PostgreSQL 的 WAL 流，每次变更以 `XLogInsert` 提交为标准 WAL 记录。记录类型覆盖目录与 meta-page 创建、文件扩展、基索引批量构建、字节级更新（单区与多区）以及 delta-base 交换，全部编码为**幂等、可重放**的形式（文件路径、offset、payload）。meta 类与 data 类记录共享同一 WAL 流、同一事务边界与 flush-on-commit 保证；meta/data 的划分只决定某条记录何时可以被写入其 mmap 支撑的页。崩溃恢复时，PostgreSQL 启动进程调用 Nova 针对 `RM_NOVA_ID` 的 redo handler，通过 `seek` 与 `write` 重放记录。整条恢复路径完全跑在 PostgreSQL 标准恢复循环内，**不需要跨系统的 flush 协调**。

**Redo 优化**：恢复或 mirror 同步时，Nova 从最近检查点开始重放日志。为降低对并发负载的影响，每条 WAL 记录带一个 type 字段：**大日志使用 direct I/O** 绕过 page cache，避免驱逐其他工作负载的缓存数据；**小日志继续用 buffered I/O** 以合并 I/O、提升 redo 效率。

### 模块 2：多租户与并发优化（Sec. 3.4）

#### 并发索引构建（CIC）

**设计动机**：多租户环境对 DDL 并发要求极高。PostgreSQL 原生的 `CREATE INDEX CONCURRENTLY` 涉及**四个阶段、两次全表扫描、多次长时间事务等待**，容易造成锁争用与 "DDL storms"。Nova 利用 delta 架构把写入卸载到 delta 文件，把流程压缩为**三个阶段**：

1. **initialization**：一个短事务更新系统目录，把索引标记为 "writable"，后续 DML 通过重定向写入 delta 文件而无需阻塞。等所有既有事务结束后：
2. **base index construction**：新事务扫描表并构建主索引，同时不阻塞并发写入。
3. **activation**：一个简短的激活事务把索引标记为 "readable"，使其对查询可用。

该协议只需**一次全表扫描**，显著降低锁争用。虽然可能引入轻微数据重复，但在查询时会被安全消解，换来了高并发环境下在线 DDL 吞吐的关键优势。

**REINDEX**：类似方法可实现 `REINDEX INDEX CONCURRENTLY`（RIC）。当数据偏离原始 IVF 质心时，磁盘驻留基索引的 recall 会下降，正确做法是重新聚类并重建。Nova 暴露漂移指标（如每簇大小偏斜、观测到的 recall 回退），由 `autovacuum` 或上游负载管理器采样，越过阈值即触发 RIC，且前台读写全程非阻塞。

#### [[Workload-Aware Pre-Warming|工作负载感知的索引预热]]

**设计动机**：消除 [[Cold Start|冷启动]] 带来的磁盘 I/O 延迟。

**核心是一个优先级模型**，把自动预测与人工干预结合。索引 $i$ 的优先级分数 $P_i$ 由下式给出（详见「关键公式」）：

$$
P_i = \alpha \cdot S_{\text{pred}} + \beta \cdot S_{\text{man}} - \gamma \cdot S_{\text{cost}}
$$

- $S_{\text{pred}}$：预测分，来自 [[ARIMA]] 时序模型的置信度——模型分析索引历史 QPS 以预测未来访问高峰；
- $S_{\text{man}}$：人工分，反映人工提交的请求，并按请求来源的历史准确率（声誉）加权；
- $S_{\text{cost}}$：代价分，与索引磁盘体积成正比，用于抑制大索引的优先级以促进公平分配；
- $\alpha, \beta, \gamma$：可调超参，分别平衡预测趋势、人工输入与资源代价的影响。

调度器按此优先队列把 Nova 索引的内存映射区域加载进物理内存（受系统资源约束）。这种数据驱动、成本感知的方法把索引预热从手工操作变成了**自动化的性能保证**。

#### [[Anytime Query|随时查询]]

**设计动机**：缓解交互式查询或冷启动下长尾请求造成的 P99 抖动。

**核心思路**：把超时控制从查询执行器**下推到物理扫描算子**（ANN 索引扫描、顺序扫描等）。达到超时时，扫描算子优雅终止并返回目前已累积的**部分结果**，而不是让整个查询失败。这种「软超时」策略保证了及时响应，用户体验优于硬失败；把超时逻辑限制在无状态的扫描层，系统也就保持了健壮性，避免了中断 join 这类有状态算子的复杂性。

### 模块 3：[[Nova Memory]]（NovaM）—— 内存图索引

**设计动机**：当数据可完全驻留内存时，[[HNSW]] 提供强性能。NovaM 以 HNSW 为核心索引，并在工程上支持高效空间管理与无锁 `scan`/`insert`/`vacuum`。

#### 索引布局（Figure 4）

- **元数据**：向量总数 $N_{vec}$、标记删除的向量数 $N_{del}$、最大 HNSW 层 $L_{max}$、入口节点 $v_0$ 等。
- **base file**：向量被切分到多个 base file，每个有固定容量；base file 数量可动态增长以扩展索引容量。
- **向量与图解耦存储**：向量数据集中存储，图结构信息（邻居节点）放在独立的 node data 区；node offset 记录每个节点的存储位置。
- **空闲节点数组**：通过 [[Free Node Array|空闲节点数组]] 管理和复用已删除节点的空间，$N_{free}$ slots 表示空闲槽位数，$fs\_id_i$ 指出空闲槽位的位置。
- **逐节点元数据**：层数 $L$、删除标志 $F_{del}$、`ctid` 列表、各层邻居 $N_i(\cdot)$。

#### 挑战一：重复向量导致的索引膨胀

生产中有两个重复向量来源：（a）应用层显式插入相同向量；（b）[[MVCC]] 机制——当非向量列被更新且**不是 HOT（Heap Only Tuple）更新**时，PostgreSQL 会创建带新 `ctid` 的新堆元组，并把写入传播到所有索引（含 ANN 索引），产生一个携带相同向量但 `ctid` 不同的新索引项。原版 HNSW 会为每个重复向量创建独立节点，导致索引膨胀与 recall 退化。

**[[Duplicate Vector Merging|重复向量合并]]**：NovaM 允许**单个节点关联多个 `ctid`**。插入时在 layer 0 检查是否有候选邻居落在预定义距离阈值内；若是且其 `ctid` 列表未满，则以无锁方式把新 `ctid` 追加到该列表并完成写入；否则在 base file 中创建一个新的物理节点。

#### 挑战二：空间复用

NovaM 的 vacuum 进程维护一个空闲槽数组，回收 `ctid` 列表为空或写入被中止的过时节点。插入时先从数组尾部尝试取一个空闲槽 ID；若成功，则**复用该节点原有的层数**而不是重新随机生成。如果这次写入最终是通过追加到已有 `ctid` 列表完成的（即所取节点因重复向量合并而未被使用），空闲节点会被归还数组。空闲槽数组的操作是线程安全的——索引扫描从不访问它，且 Nova 强制 `flush` 与 `vacuum` 互斥。

#### Vacuum 的三阶段

从图索引中删除节点是昂贵的，因为入边可能散布全图。NovaM 用**三种节点状态**做批量回收：

| 状态 | 含义 | 可被遍历 | 可作为邻居 | 可被复用 |
|---|---|---|---|---|
| **active** | 正常图节点 | ✓ | ✓ | — |
| **deleted** | vacuum 后 `ctid` 列表为空的节点，暂时保留以维持图连通性 | ✓（仅用于查询路由） | ✗ | ✗ |
| **free** | 已从图中移除，等待复用 | ✗ | ✗ | ✓ |

三个阶段：

1. **`remove_heap_ctids`**：从 active 节点移除死 `ctid`；若某节点列表变空则标记为 deleted。同时记录最高的 active 节点（不含入口点），以备入口点更新。
2. **`repair_graph`**：若 deleted 节点数超过阈值，扫描 active 节点、移除指向 deleted 邻居的边，并调用 `repair_node` 重建受影响的邻居列表。
3. **`mark_free`**：全局把已回收节点标记为 free，防止并发查询在复用后访问它们。

vacuum 结束时，deleted 节点从图中移除并加入空闲槽数组。

#### 无锁设计（三个并发场景）

1. **`remove_heap_ctids` vs `scan`**：在节点内用**原地双指针 + 原子操作**过滤死 `ctid`——左指针寻找死 `ctid`，右指针寻找活 `ctid`，把右指针指向的活元素原子复制到左指针位置，最后原子更新 `ctid` 列表长度。该过程可能**临时对并发查询暴露重复 `ctid`**，但在合并 delta 与 base 结果时会被安全消除，因此最终返回上层的 `ctid` 仍然无重复。
2. **`repair_node` vs `query`**：`repair_node` 生成一份全新的完整邻居列表，并逐个原子替换旧邻居，避免查询访问到非法边。此过程中查询可能同时看到新老邻居，但所有读到的邻居 ID 都指向有效节点且确实是原节点的邻居，因而保住了图连通性、不明显损害召回质量。
3. **`mark_free` vs `query`**：`mark_free` **只把空闲节点的槽位加入空闲数组，不清空节点内容**。同时在合适位置放置内存屏障，实现 release-acquire 语义，保证时序正确性。

### 模块 4：[[Nova Disk]]（NovaD）—— 磁盘混合索引

**设计动机**：NovaD 面向磁盘驻留、大规模、多租户场景——索引体积超过可用内存，且扫描、插入、vacuum 在高并发下混合发生。作者**选择 IVF 而非磁盘图索引**（如 [[DiskANN]]），理由是：IVF 的簇局部组织带来**有界的单查询 I/O 与基本顺序的访问**，并且天然适配按租户内存预算做**按需簇加载**；而磁盘图索引的重建代价、内存 PQ/热点节点驻留需求、合并期写放大以及随机 I/O 驱动的 P99 长尾，都与 NovaD 必须遵守的每租户内存与稳定性预算相冲突。

#### 索引布局（Figure 5）

**两层结构**：fast layer（质心导航）+ base layer（向量存储）。

- **fast layer**：紧凑、cache 友好的布局存放 $C$ 个簇质心 $\text{vec}_1, \ldots, \text{vec}_C$。视质心规模，该层**可选地叠加一个图索引**——启用时，质心列表后紧跟对应的邻接表 $N(v_1), \ldots, N(v_C)$，以加速子空间定位。
- **metadata**：向量总数 $N_{vec}$、簇数 $C$、单 base file 最大容量 $N_{max}$、各簇大小 $S_1, \ldots, S_C$ 等。
- **base layer**：避开朴素的「一簇一文件」设计（大 $C$ 下会制造大量小文件、损害文件系统性能）。所有向量存放在**共享 base file** 中，每簇最多获得 $N_{max}/C$ 个槽位；当某簇在最后一个 base file 中的配额耗尽时，`expand` 为该簇的溢出**专门创建新的 base file**，其他簇的槽位保持逻辑分配但物理不写入，避免存储浪费。向量数据与关联 `ctid` 存放在**分开的连续区域**以改善顺序访问与 cache 局部性。

#### 关键操作

- **`scan`**：先遍历 fast layer 找出质心与查询向量接近的候选簇，再定向遍历这些簇内的向量。每个簇在 base file 中的起始偏移是**静态确定**的，结合其当前大小 $S_i$ 就能无歧义地推导出要读的文件区段——因此扫描单个簇只产生**有界的随机 I/O**，主要发生在 base file 之间的切换处。$S_i$ 的读取是原子的，并且会对 base file 中向量的可见性做校验，从而在并发 `insert` 与 `vacuum` 下保持一致。
- **[[Adaptive Scan|自适应扫描]]**：用户可通过 `nprobes`（检查的簇数）等参数平衡延迟与 recall。自适应机制进一步根据**查询向量与候选质心的距离分布**动态校准遍历范围：若到多个质心的距离彼此相差在 **20%** 以内，推断查询向量位于簇边界附近，则穷尽扫描所有邻近簇以保住 recall；反之若某个质心在邻近性上占绝对主导，则推断查询落在该簇内部，**提前终止**以削减 I/O。
- **`insert`**：fast layer 先解析目标簇；借助当前簇大小 $S_i$ 算出确切槽位并写入向量，随后**原子自增 $S_i$，全程无锁**。若插入使该簇在当前 base file 中达到槽位容量，则无缝调用 `expand` 为该簇准备新的 base file。
- **`vacuum`**：扫描 base file 条目识别逻辑删除的向量，用**双指针技术**记录源条目及其目标位置（目标可能位于不同 base file）。重定位时先把目标槽标记为不可见，再复制源条目并使其可见，最后使源槽失效。压实完成后更新 $S_i$，并立即删除完全废弃的 base file。注意：vacuum 的某些阶段可能让**瞬时重复条目**变得可观测，但正确性由 `scan` 期间的去重保证。

### 模块 5：高级查询处理

#### 混合查询的四种计划类型（Figure 6）

Nova 采用与 [[ADB-V]] 类似的**基于代价模型的方法**，从多个候选计划中择优选型。在 ADB-V 的分类基础上，Nova 支持四类计划：

| 类型 | 名称 | 流程 | 特点 |
|---|---|---|---|
| **Type 1** | Plan A / Brute-force | 扫描所有满足标量谓词的行，计算到查询向量的距离，返回 top-$k$ | 无索引参与 |
| **Type 2** | Plan B/C、[[Bitmap Filtering|位图预过滤]] | 先构造满足标量谓词的行 ID **bitmap**，再在向量索引扫描中用它跳过不合格向量 | 显式物化位图 |
| **Type 3** | Plan D、后过滤 | 先用向量索引取出 $\sigma \times k$ 的候选集（$\sigma$ 为 [[Recall Amplification Factor|放大因子]]），再对候选验证标量谓词 | 先召回后过滤 |
| **Type 4** | **On-the-fly Filtering（本文引入）** | 保持 Type 2 的**预过滤语义**（只有满足标量谓词的向量才被允许进入索引扫描），但把谓词表达为**回调式表达式求值器**，在向量索引遍历中逐候选 `ctid` 调用 | 避免 bitmap 构造 |

**Type 4 与 Type 2 的区别**只在于谓词的物化方式：Type 2 提前构造显式 bitmap，Type 4 用回调。**Type 4 与 Type 3 的区别**则是本质性的——Type 3 是后过滤计划，先从索引取 $\sigma \times k$ 个候选、再对它们求值标量谓词。

#### [[Iterative Query|迭代查询]]

**动机**：Type 3 的静态放大因子 $\sigma$ 往往无法适应运行时数据分布——过滤后可能拿不满 $k$ 个结果。

**做法**：NovaM 借助图索引的**增量遍历特性**（受 [[VBASE]] 启发）实现迭代机制。算法先生成一个小的候选批次；若标量过滤后得到的结果少于 $k$，则通过探索相邻节点逐步扩张搜索前沿，直到满足要求或触发停止条件。论文明确：这是 Type 3 后过滤流水线的**精化**，而非新计划类型，也与 Type 4 的回调预过滤相互独立。

#### [[Partial Index|部分索引]] 与 [[Covering Index|覆盖索引]]

- **Partial Index**：生产负载中查询常命中**热点谓词**。Nova 支持在这些谓词定义的子集上建部分索引。若某查询完全落在某个部分索引内，就直接在受限索引上执行向量搜索，从而收窄搜索范围；若有多个部分索引可用，Nova 选择**最具体（即子集最小）**的那个来执行。
- **Covering Index**：真实应用中高频查询常需在特定结构化字段上过滤——例如 Model Studio 的 RAG 场景几乎**所有查询**都要按 document ID 过滤。受关系数据库启发，Nova 支持把高频查询的标量属性**物化进 ANN 索引结构内部**（如 NovaM 的图节点内），使过滤可在索引扫描期间完成，从而免除额外的堆访问、显著降低 I/O。

#### 计划选择策略

Nova 采用**规则启发式 + 精化代价模型**的混合选择策略：

1. **部分索引重写优先**：若部分索引覆盖查询谓词，查询被重写为针对该缩减索引的纯向量或混合检索。
2. **纯向量检索**：在暴力扫描与索引扫描之间选择，取决于预期检索量 $(k \times \sigma)$ 是否占总基数中显著比例。
3. **混合检索**：Nova 在 ADB-V 代价模型基础上引入标量选择率、ANN 扫描代价、距离计算代价、堆访问代价与 bitmap 构造开销等因素。
4. **迭代查询**：模型进一步计入每轮迭代代价与满足 $k$ 所需的预期迭代次数。

#### 跨表混合查询（Sec. 5.2）

RAG 中文档与 chunk 是一对多关系，典型查询在 `t_embedding` 上做向量检索、同时按 `t_metadata` 的元数据过滤，两表以 `doc_id` 连接（示例 SQL 见 Sec. 6.3）。传统优化器在过滤选择率变化时常常生成不稳定的执行计划。Nova 提出两个候选策略，并用**基于代价的自适应策略**在三者间动态选择。

**策略一：[[ANN-driven Hash Join|ANN 驱动的 Hash Join]]**

优先做向量检索：先在 embedding 表上执行大规模 ANN 扫描，用结果构造哈希表，再用过滤后的 metadata 表逐行探测。性能由初始 ANN 扫描规模主导，代价模型见「关键公式」。

**策略二：[[Metadata-driven Join|元数据驱动的 Join]]**

先应用元数据过滤，把过滤后的 metadata 表与 embedding 表在 `doc_id` 上连接，取出所有合格文档的 chunk，再对这些 chunk **精确计算**向量距离以产出 top-$k$。连接步骤由 ADB-PG 在 Nested Loop、Hash Join、Merge Join 等物理算子中选择。论文分析 Nested Loop 变体，因为**在元数据过滤高度选择性时（$N_f$ 小）它主导整体代价**——而这正是该策略优于 ANN 驱动 Hash Join 的区间。

**自适应选择**：优化器比较 $Cost_{HJ}$ 与 $Cost_{NL}$。元数据过滤**选择性弱（$N_f$ 大）时选 Hash Join**，因为一次大规模 ANN 扫描比多次小查找更便宜；过滤**选择性极强（$N_f$ 小）时选元数据驱动 Join**，因为外循环次数少、内层扫描更经济。该策略保证了稳定高效的查询计划生成，缓解了选择率波动与 ANN 代价难以准确估计所带来的性能抖动。

---

## 关键公式

### 公式 1：[[Workload-Aware Pre-Warming|索引预热优先级]]

$$
P_i = \alpha \cdot S_{\text{pred}} + \beta \cdot S_{\text{man}} - \gamma \cdot S_{\text{cost}}
$$

**含义**：为每个 Nova 索引计算预热优先级分数，综合「预测的未来访问热度」「人工指定」「资源代价」三个归一化分数，决定谁先被加载进内存。

**符号说明**：

- $P_i$：索引 $i$ 的优先级分数
- $S_{\text{pred}}$：预测分，来自 [[ARIMA]] 时序模型对其历史 QPS 分析的置信度
- $S_{\text{man}}$：人工分，由人工提交的请求构成，按请求来源的历史准确率（声誉）加权
- $S_{\text{cost}}$：代价分，与索引磁盘体积成正比，用于抑制大索引、促进公平分配
- $\alpha, \beta, \gamma$：可调超参，分别平衡预测趋势、人工输入与资源代价

### 公式 2：[[ANN-driven Hash Join|ANN 驱动 Hash Join 的代价模型]]

$$
Cost_{HJ} \propto \min\left(k \cdot \sigma \cdot \frac{N}{N_f},\ S_{max}\right)
$$

**含义**：该计划的性能由初始 ANN 扫描的规模主导；扫描量随「逆选择率」线性增长，但被系统上限截断。

**符号说明**：

- $k$：查询的 top-$k$ 限制
- $\sigma$：ANN 扫描的放大因子
- $N$：embedding 表中的总行数
- $N_f$：满足元数据过滤的估算行数
- $N/N_f$：**逆过滤选择率**（inverse filter selectivity）
- $S_{max}$：系统定义的可扫描点数上限，用于截断代价

### 公式 3：[[Metadata-driven Join|元数据驱动 Join 的代价模型]]

$$
Cost_{NL} \propto N_f \times R_{ppi}
$$

**含义**：该策略的代价由内表访问总次数决定——外循环中每个合格文档都要按平均 chunk 数做一次内表查找。

**符号说明**：

- $N_f$：外循环中合格文档数
- $R_{ppi}$：估算的**每文档平均 chunk 数**（per-document pieces），代表每次查找的工作量

> 两式构成自适应选择的基础：$N_f$ 小则 $Cost_{NL}$ 小，选元数据驱动 Join；$N_f$ 大则 $N/N_f$ 小、$Cost_{HJ}$ 小，选 Hash Join。

---

## 关键图表

> 共 16 个 Figure、1 个 Table，全部按原论文编号完整保留。

### Figure 1：AnalyticDB-PG 总体架构

![[Nova_fig1_arch_adbpg.png]]

展示 [[AnalyticDB-PG|ADB-PG]] 的 coordinator（Parser、QD、DTM、Optimizer）+ 多个 worker（各自持有 table、index、WAL、Executor）的 [[MPP Architecture|MPP]] 结构，Nova 即构建于此之上。

### Figure 2：Alibaba Cloud Model Studio 总览

![[Nova_fig2_model_studio.png]]

文档被切分为 chunks 并嵌入为 embeddings，连同元数据（filename、tags 等）分别写入 `t_embedding` 与 `t_metadata`；[[RAG]] 查询先做 Filtered Vector Retrieval，再把结果连同 prompt 送入通义（Tongyi）大模型。

### Figure 3：Nova 索引框架总览

![[Nova_fig3_framework.png]]

Base index（Metadata file + Structure file(s)）与多个 delta file、WAL 共同构成索引；`Scan`/`Insert`/`Flush`/`Vacuum` 四种操作作用于其上。delta file 内部 `ctid` 与 `vec` 分区连续存储，便于 SIMD 等硬件加速。

### Figure 4：Nova Memory 结构

![[Nova_fig4_memory_structure.png]]

metadata file 记录 $N_{vec}$、$N_{del}$、$L_{max}$、入口节点 $v_0$；base file 按固定容量存向量；node offsets 记录每节点存储位置，并通过空闲槽数组（$N_{\text{free slots}}$、$fs\_id_i$）管理回收空间；node header 存层数 $L$、删除标志 $F_{del}$、`ctid` 列表及各层邻居 $N_i(\cdot)$。

### Figure 5：Nova Disk 结构

![[Nova_fig5_disk_structure.png]]

fast layer file 存质心 $\text{vec}_1, \ldots, \text{vec}_C$ 及可选邻接表 $N(v_i)$；metadata file 存 $N_{vec}$、$N_{max}$、$C$ 与各簇大小 $S_1, \ldots, S_C$；base layer 为共享 base file，按簇分区存放 `ctid` 与向量，簇满后通过新 base file 扩容。

### Figure 6：Nova 混合查询框架

![[Nova_fig6_hybrid_query_plans.png]]

Type 1（暴力扫描）、Type 2（标量索引扫描 + bitmap + 向量索引扫描）、Type 3（向量索引扫描取 top-$(\sigma \times k)$ 后标量过滤）、Type 4（构造回调函数 + 向量索引扫描）四种计划的数据流对比。

### Figure 7：GIST/Cohere/Tongyi 上的整体性能

![[Nova_fig7_overall_perf.png]]

三个数据集（GIST-1M、Cohere-10M、Tongyi-10M）× 四个指标（QPS、平均延迟、P99 延迟、写延迟）的 12 张子图。[[Nova Disk|NovaD]] 与 [[Nova Memory|NovaM]] 在几乎所有 recall 目标下取得最高吞吐与最低 P99；例如 GIST-1M 在 95–96% recall 时 NovaD 达 5000+ QPS、P99 8.2ms，约为 pgvector 吞吐的 4.2 倍、延迟的 1/2.2，为 Milvus 吞吐的 4.8 倍、延迟的 1/1.5。

### Figure 8：索引构建时间

![[Nova_fig8_index_build_time.png]]

GIST-1M 与 Cohere-10M 上各系统的 Write + Index Build 分段耗时。NovaD 的索引构建时间仅占总耗时的 12%–14%，在 GIST-1M / Cohere-10M 上分别比最慢的 pgvector 快 14.8×/15.8×，比 Milvus 快 4.0×/3.3×。

### Figure 9：存储占用对比

![[Nova_fig9_storage_footprint.png]]

GIST-1M 上 NovaD 索引仅 408MB（约为 ADB-V 的 1/12、pgvector 的 1/18）；Cohere-10M 上 NovaD 为 3.0GB，而 ADB-V/pgvector/Milvus 分别需要 44GB/39GB/118.95GB。层次化量化编码大幅压缩存储同时保持检索质量。

### Figure 10：多租户活跃租户数的影响

![[Nova_fig10_tenant_count.png]]

10/100/300 张表场景下 QPS、mRT、P99RT、MaxRT 的对比（recall ≥ 95%）。随活跃租户数增加，各系统 QPS 下降、延迟上升（内存换入换出更频繁），但 NovaD 在所有租户规模下全面领先。

### Figure 11：过滤选择率对跨表混合检索的影响

![[Nova_fig11_selectivity.png]]

None/99%/50%/1% 四档选择率下 QPS、mRT、P99RT、MaxRT 的对比。NovaD 在所有选择率设置下吞吐最高、延迟最低，印证其跨表混合检索在选择率波动下的鲁棒性。

### Figure 12：CIC（并发建索引）消融——构建时间与写 TPS

![[Nova_fig12_cic_ablation.png]]

(a) Create Index (CI) vs. Create Index Concurrently (CIC) 构建耗时：pgvector 的 CIC 因等待最长事务（$T_{long}$）高达 7859s，NovaD 的 CIC 仅 80s（几乎与全量构建的 84s 持平）；(b) 索引构建完成后 vs. CIC 期间的写 TPS，NovaM/NovaD 均保持约 1100–1250 TPS。

### Figure 13：NovaM 与 ADB-V 的读写并发对比

![[Nova_fig13_rw_concurrency.png]]

(a) NovaM 在并发场景下 QPS 稳定在约 5000–6000，P95/P99 延迟即便在 flush 期间也保持平稳；(b) ADB-V 由于 HNSW 图上的页级锁争用，QPS 出现多次塌陷，尾延迟出现尖峰。

### Figure 14：Delta 文件大小的影响

![[Nova_fig14_delta_size.png]]

Delta 大小从 1000 增至 10000 向量时，QPS 从约 6400 降至约 3700，P99 RT 明显上升，印证了 Nova 自动 flush 机制（默认阈值 5000 向量，由写热度与大小阈值共同驱动）的必要性。

### Figure 15：插入 TPS

![[Nova_fig15_insert_tps.png]]

16 个插入线程、10M 向量数据集上，`NovaM w/o index`（无索引上界）与 `NovaM`（含索引）曲线接近——因为 delta 文件吸收写入、基索引维护异步进行；而 ADB-V 远远落后，因为每次插入都要在细粒度锁下竞争图结构。

### Figure 16：Model Studio 生产环境 24 小时对比

![[Nova_fig16_24h_production.png]]

(a) ADB-V 在 3:00 AM 大批量摄取触发的 vacuum 期间，Query/Upsert 的 P99 延迟剧烈尖峰（共享缓冲池被污染、页级锁与并发查询争用）；(b) Nova 全程保持稳定——得益于 Delta-Base 架构把写冲突限制在 delta 文件内，使基索引查询无锁，vacuum 不阻塞前台查询。

### Table 1：CIC 对 Top-100 Recall 的影响（GIST-1M）

| System | Create Index | Create Index Concurrently |
|---|---:|---:|
| NovaM | 99.14% | 99.17% |
| NovaD | 99.19% | 99.19% |

**说明**：Nova 的三阶段并发建索引协议几乎不损失 recall，验证了「高并发 DDL 吞吐」与「检索质量」可以兼得。

---

## 实验与证据

### 数据集

| 数据集 | 规模 | 维度 | 来源 | 用途 |
|---|---:|---:|---|---|
| [[GIST Dataset]]-1M | 1M | 960 | 公开 ANN benchmark | 单表混合/纯向量检索 |
| [[Cohere Dataset]]-10M | 10M | 768 | Cohere embedding 公开数据集 | 单表混合/纯向量检索 |
| [[Tongyi Dataset]]-10M | 10M | 1536 | 阿里通义（内部） | 单表混合/纯向量检索 |
| Model Studio 生产流量 | 40 万+ 用户、60 万+ 数据库、1000 亿+ 向量 | — | 真实业务 | 24 小时生产对比 |

### 实验设置

- **硬件**：Alibaba Cloud ECS g8i 系列实例。
- **对比系统**：[[Milvus]]、[[PGVector|pgvector]]、[[Elasticsearch|ES]]、[[ADB-V]]（Nova 前身）、PostgreSQL-V；均在相同 recall（≥95%）约束下比较 QPS / 平均延迟 / P99 延迟 / 写延迟。
- **评测框架**：[[VDBBench|VectorDBBench]] 默认单表配置用于整体性能；自定义多租户与跨表测试用于验证生产特性。

### 关键结果

1. **整体性能**（Figure 7）：NovaD/NovaM 在三套数据集的全部四项指标上普遍领先，GIST-1M 上吞吐可达 pgvector 的 4.2×、Milvus 的 4.8×。
2. **构建与存储效率**（Figure 8、9）：索引构建时间比 pgvector 快 14–16×，比 Milvus 快 3–4×；存储占用在 Cohere-10M 上仅为 ADB-V/pgvector 的 1/15–1/13、Milvus 的 1/40。
3. **多租户鲁棒性**（Figure 10）：租户表数从 10 增到 300，NovaD 的 QPS 衰减幅度与延迟增长幅度均显著小于对比系统。
4. **跨表混合检索**（Figure 11）：在 None/99%/50%/1% 四档选择率下，NovaD 全面优于 Milvus/ADB-V/pgvector/ES，验证了基于代价的自适应 Join 策略的有效性。
5. **并发建索引（CIC）**（Figure 12、Table 1）：pgvector 的 CIC 因等待长事务耗时近 2.2 小时，NovaD 仅 80 秒；recall 损失可忽略（99.14%→99.17%，99.19%→99.19%）。
6. **读写并发稳定性**（Figure 13）：NovaM 无锁设计使 QPS 与尾延迟在 flush 期间保持平稳，ADB-V 因页级锁争用反复出现 QPS 塌陷。
7. **写入吞吐**（Figure 14、15）：delta 文件把插入延迟与基索引维护解耦，插入 TPS 接近「无索引上界」，远超 ADB-V。
8. **生产验证**（Figure 16）：在 Model Studio 真实 24 小时流量下，ADB-V 在凌晨批量摄取引发 `vacuum` 期间出现 P99 尖峰，Nova 全程平稳，直接印证 delta-base 架构解决了生产痛点。

---

## 批判性思考

### 优点

1. **问题驱动而非指标驱动**：论文从 Model Studio 实际运维痛点（多租户内存压力、写入抖动、跨表 RAG 查询）出发设计系统，而非单纯追求 ANN benchmark 上的 recall-QPS 曲线，Figure 16 的真实 24 小时对比是比大多数向量数据库论文更有说服力的证据。
2. **Delta-Base 架构的工程完整性**：把写写冲突收敛到 delta 文件、让基索引可以无锁扫描，同时用两级 WAL 解耦 meta/data 持久化顺序，是一套自洽且可落地的并发控制方案，三种无锁并发场景（`remove_heap_ctids`/`repair_node`/`mark_free` vs 查询）的论证也相对具体。
3. **跨表混合检索的系统化处理**：多数向量数据库论文只讨论单表标量过滤，本文把 RAG 常见的「文档-chunk」两表模式显式建模为 ANN-driven Hash Join 与 Metadata-driven Join 的代价权衡，填补了实际空白。
4. **CIC 的务实设计**：三阶段把 PostgreSQL 原生 CIC 的"两次全表扫描+长事务等待"压缩为一次扫描，recall 代价可忽略，直接解决多租户环境下 DDL 风暴问题。

### 局限性

1. **未与专用磁盘图索引正面对比**：论文论证选择 IVF 而非 [[DiskANN]] 类磁盘图索引，但实验部分未给出 NovaD vs 优化后的磁盘图索引（如论文一开始提及的工业级 DiskANN 变体）在相同内存预算下的直接数据，选择的合理性更多靠论证而非量化对比。
2. **「随时查询」与「迭代查询」的量化效果缺失**：Anytime Query（软超时部分结果）与 Iterative Query（动态放大因子）两个特性在 Sec. 3/5 均有详细设计说明，但实验部分（Sec. 6）没有专门的消融实验量化其对 P99/recall 的边际贡献。
3. **代价模型参数缺乏公开细节**：预热优先级公式中的 $\alpha, \beta, \gamma$ 以及混合查询代价模型中的权重系数，论文未给出具体取值或调参方法论，复现时需要自行摸索。
4. **量化方案的 recall 权衡讨论较简略**：论文提到支持 PQ、RaBitQ 等量化但原始向量仍保留在 delta 文件，对量化引入的 recall 损失与存储/延迟收益之间的量化关系（例如不同量化率下的 recall-QPS 曲线）着墨不多。
5. **mmap 依赖 POSIX 文件系统**：当前实现绑定本地 NVMe/ESSD，向对象存储（OSS）的扩展被明确列为未来工作，这意味着目前的方案尚不支持计算存储完全分离的云原生部署形态。

### 可借鉴方向

- 若设计多租户向量服务，delta-base 分层（写入隔离在小文件、查询下沉到无锁基索引）比直接复用关系数据库页锁模型更适合吸收突发写负载。
- 若需要兼顾实时写入可见性与批量索引效率，可参考 Nova 的「写延迟与可见性解耦」思路：先写轻量结构保证可见、再异步合并到主索引。
- 若涉及多表混合检索（如 RAG 的文档-chunk 模式），应把"先过滤再向量"与"先向量再过滤"都纳入代价模型，而不是固定选择其一。
- CIC 的"写入重定向 + 单趟扫描 + 简短激活事务"三段式协议，可推广到任意需要在线构建的索引结构，用以降低多租户 DDL 的锁争用。

---

## 关联笔记

### 基于
- [[ADB-V]]：Nova 的前身，Nova 延续其基于代价模型的混合查询选型思路，并直接针对其页锁争用、vacuum 期间延迟尖峰等痛点做了架构性改进。
- [[HNSW]]：NovaM 的底层图索引算法，Nova 在其上做了重复向量合并与无锁空间复用。
- [[Inverted File Index]]：NovaD 的底层聚类索引算法，Nova 在其上做了共享 base file 布局与自适应扫描。

### 对比
- [[Milvus]]、[[PGVector]]、[[Elasticsearch]]：论文核心基线，分别代表段式组织、页式关系数据库集成、倒排+向量混合三类范式。
- [[GaussDBVector]]：同为"把磁盘 ANN 索引整合进关系型数据库"的工作，可与 Nova 对比图索引（Vamana）vs 聚类索引（IVF）在磁盘驻留场景下的设计取舍。
- [[LindormVector]]：同为云原生分布式向量引擎，可用于对比"关系数据库内核整合 + 多租户"与"多模 NoSQL 内建向量服务"的架构差异。

### 方法相关
- [[Delta File]] / [[Ring Queue]]：Nova 写入路径的核心数据结构。
- [[WAL]]：Nova 两级（meta/data）持久化顺序设计的基础机制。
- [[MVCC]]：解释了为何非 HOT 更新会放大重复向量问题，是 NovaM 重复向量合并设计的根本动机。
- [[Cost-Based Adaptive Join|Cost-Based Join Selection]]：跨表混合检索中 ANN-driven Hash Join 与 Metadata-driven Join 之间的自适应切换机制。
- [[Hybrid Search]]：Nova 混合查询四种计划类型（暴力、bitmap 预过滤、后过滤、回调预过滤）所属的更广泛主题。
- [[RAG]]：Model Studio 场景下跨表混合检索的下游应用。

### 硬件/数据相关
- [[VDBBench|VectorDBBench]]：Nova 性能对比使用的标准评测框架。

---

## 速查卡片

> [!summary] Nova
> - **核心**：阿里云 AnalyticDB-PG 团队面向多租户 RAG 场景打造的向量引擎，用 delta-base 架构把写入与基索引解耦。
> - **索引**：NovaM（HNSW + 重复向量合并 + 无锁空间复用）、NovaD（IVF + 共享 base file + 自适应扫描）。
> - **并发**：三阶段并发索引构建（CIC）、两级 WAL（meta/data 分离刷盘顺序）、工作负载感知预热、随时查询（软超时）。
> - **查询**：四类混合查询计划（含新引入的回调式预过滤 Type 4）+ 跨表混合检索的代价自适应 Join 选择（ANN-driven Hash Join vs Metadata-driven Join）。
> - **证据**：13 页论文含 16 图、1 表；GIST/Cohere/Tongyi 上吞吐领先基线 3–5×，Model Studio 24 小时生产压测验证消除了前身 ADB-V 的 vacuum 期间延迟尖峰。

---

*笔记创建时间: 2026-10-08*
