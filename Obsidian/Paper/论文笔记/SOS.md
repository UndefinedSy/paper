---
title: "SOS: A High-Performance Distributed Key-Value Store for Large-Scale Online Services"
method_name: "SOS"
authors: [Yingxin Li, Kai Liu, Han Xie]
year: 2026
venue: PVLDB 19(12)
tags: [key-value-store, distributed-storage, hash-index, chain-replication, two-phase-commit, write-amplification, llm-inference, kv-cache]
zotero_collection: VLDB/2026
image_source: local
created: 2026-10-10
---

# 论文笔记：SOS: A High-Performance Distributed Key-Value Store for Large-Scale Online Services

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | 腾讯（Tencent Inc.，北京） |
| 日期 | 2026，PVLDB 19(12): 4237–4249 |
| 项目主页 | https://github.com/holmes1412/sos-vldb2026-supplementary |
| 对比基线 | [[RocksDB]] |
| 链接 | [DOI](https://doi.org/10.14778/3827998.3828029) |
| 原始 PDF | `VLDB/2026/SOS (KV Store Tencent).pdf` |

---

## 一句话总结

> SOS（Sogou Open Storage）用"内存哈希索引 + 变长分级磁盘布局 + 异步写缓存"三件套，保证每次读写操作最多一次物理磁盘 I/O，在 SSD 成本上实现接近内存存储的延迟，服务腾讯超 50PB、7 万亿条记录的生产集群。

---

## 核心贡献

1. **至多一次磁盘 I/O 的设计不变式**：通过三个协同设计的子系统（内存哈希索引、分级 block-chunk 磁盘引擎、异步写缓存）共同保证任意 GET/PUT/DELETE 操作最多触发一次物理磁盘 I/O，从根本上消除 [[LSM-tree]] 式的读写放大。
2. **分级 Block-Chunk 磁盘存储引擎**：用 step-sequence 变长 block 分配方案，把内部碎片控制在 ≈7.93%，远低于传统 2 的幂次分配的 25%–30%；配合 chunk 粒度的写缓存和异步 flush，实现 1.00× 的写放大。
3. **Primary-Based Chain Replication with Two-Phase Commit**：把失败率高的操作（校验、磁盘写）放进 Phase 1，把轻量操作（索引提交）放进 Phase 2，大幅降低提交阶段失败概率；配合 in-chain retry 机制，把瞬时抖动下的写失败率降到接近零。
4. **十余年生产验证**：自 2013 年起持续演进，目前在腾讯 40+ 生产集群部署超 50PB、7 万亿条记录，峰值 550M QPS，支撑元宝 RAG、混元 KV-cache、用户画像等服务，P99.9 读延迟低于 5ms，相对内存存储 TCO 降低 84%。

---

## 问题背景

### 要解决的问题

现代互联网与 AI 服务（用户画像/推荐、[[RAG]] 式搜索、Transformer [[KV Cache]] offloading）共同要求：**海量容量**（数十到数百 PB）、**严格尾延迟**（P99.9 单位数毫秒）、**高写吞吐**（周期性全量刷新）、**成本敏感**（PB 级规模下内存存储代价过高）。现有系统无法同时满足这四点。

### 现有方法的局限

- **内存存储**（[[Redis]]、[[Memcached]]）延迟够低但每 TB 成本约为 SSD 方案的 10 倍，PB 级下不可承受。
- **基于 [[LSM-tree]] 的磁盘 KV 系统**（[[RocksDB]] 等）天生带有写放大，直接转化为更高尾延迟和混合负载下更低的吞吐。
- **分布式文件/对象存储**（[[HDFS]]、[[HBase]]）为顺序/批量访问优化，无法满足毫秒级随机访问 SLA。

### 本文的动机

这些工作负载的访问模式高度一致：**点读写、无局部性、周期性大批量写入、无 range-scan 需求**。这使得哈希索引 + SSD 驻留成为自然架构选择——牺牲 key 排序换取 $O(1)$ 点访问是完全可接受的权衡，因为推荐特征检索、RAG 文档检索、KV-cache 读回都是精确 key 查找而非范围遍历；而且即便是分布式 LSM-tree 系统在跨 shard 场景下也不提供全局 key 排序，所以排序能力在分布式部署中并无实际收益，却仍要承担其全部放大成本。关键是协同设计哈希索引、读缓存与异步写缓存，使三者对磁盘状态保持一致视图——只有当所有层都认同每条记录的版本时，系统才能保证每次操作最多一次物理 I/O。

---

## 方法详解

### 整体架构

SOS 是全局无序（globally unordered）的 KV 存储系统，不同业务数据通过 service name 在统一集群内逻辑隔离而非物理隔离。每个 key 只有一个版本，带写时间戳和 TTL。集群由一个或多个 ConfigServer（CS）与 $N$ 个 DataServer（DS）组成，每个 shard 存 $R$ 个副本（1 primary + $R-1$ follower）。$R$ 在集群初始化时固定，$N$ 可动态变化，使集群容量和吞吐线性扩展。ConfigServer 管理路由表与副本分布但不在关键路径上；DataServer 负责一致性、索引、缓存与磁盘存储。

命令集涵盖 PUT、DELETE（涉及全部 $R$ 副本同步）、GET、TEST（单副本点查）、MGET（本地批量检索）、PUSH/POP（跨节点批量预取检索）、SCAN（带时间戳的全量遍历）。论文主要围绕 PUT、DELETE、GET、TEST 四个核心命令展开读写路径设计。

单次请求流程（Figure 3）：客户端本地缓存路由表，直连目标 DataServer；版本不匹配时触发表刷新。GET/TEST 直接查询 primary DataServer；PUT/DELETE 携带链元数据发往 primary，primary 本地持久化、打时间戳后转发给链上下一个 follower（细节见 §2.4 两阶段链提交）。节点内部：TEST 只读内存索引条目（存在性、时间戳、TTL），不触碰磁盘；GET 先探测读缓存，未命中则查索引得到磁盘位置，读取 KV 载荷与记录元数据后更新读缓存；PUT/DELETE 先探测索引——覆盖写/删除都要移除旧磁盘 block 及索引条目、失效读缓存条目，再插入新 block 并更新索引。

由于 KB 级 SSD 读约 700μs，而数据中心内网络 RTT 约 1ms，这一"至多一次磁盘 I/O"架构使 SOS 在 SSD 成本下达到可比拟分布式内存存储的延迟。删除/覆盖的 block 在写时同步回收，无需 compaction 或垃圾回收，从而保证稳定的尾延迟。

### 核心模块 1：Disk Storage（dstorage）

**设计动机**：如何在最小化磁盘空间浪费、简化管理、高效回收删除空间的前提下管理变长数据。

**Block、Chunk 与 Step**：

- 最小存储单元是 **block**，存一个 KV 条目及其元数据（service ID、timestamp、TTL）。dstorage 为每个条目选择能容纳它的最小 block 规格，小 KV 不继承大 size class 的空间开销。
- **Step 序列**决定 block 规格集合（见公式 1），比传统 2 的幂次分配提供更细粒度，尤其有利于小对象（大多数分配发生在这里）。
- 所有数据以 **chunk** 为单位管理（如 64MB），每个 chunk 带 header（block_size、occupancy bitmap）。磁盘初始切为若干 chunk；每个 chunk 只存一种 block 规格，但按需分配给某个 step 而非永久绑定。chunk 内全部 block 被删除后立即回收复用。对删除模式不规则导致 chunk 稀疏占用的场景，SOS 提供在线 compaction 工具供运维手动整理。

**最小 block_size 约束**：受索引可寻址性限制——32-bit block offset、2TB 分区下，实际最小值为 512 字节。对 LLM KV-cache，attention tensor 大小由模型架构固定，相应设置 block_size 可将内部碎片降到近零。

**Write Cache**：SOS 支持 Direct IO 并实现自己的类 pagecache 写回管理机制。写请求由专用后端线程负责，当某 chunk 脏块占比超过阈值即触发 flush；生产环境启用异步 flush，client 可见延迟不等待设备持久化。写缓存管理以 chunk 粒度运作：某 step 同时最多一个活跃 chunk 在内存中；无活跃 chunk 时新分配，数据全删后回收。该机制的主要代价是分布式不一致风险——进程异常退出或机器宕机时节点可能有未 flush 数据，SOS 通过把该节点从路由表移除、等待从其他副本修复来解决，年度生产运行未观测到因写缓存不完整导致的数据丢失。

**Multiple Disk Management（mdstorage）**：单 DataServer 管理多个物理磁盘（每盘一个独立 dstorage 实例），磁盘级隔离保证单盘 I/O 错误不影响其他盘；DataServer 将受影响 block 重映射到健康盘，仅在某盘错误率超阈值时才标记自身不健康，背景 flush 线程可独立运行于每块盘，提升总写吞吐。

### 核心模块 2：In-Memory Hash-Based Index

**哈希索引选择**：相比 [[LSM-tree]] 和 [[B+ Tree|B+ 树]]，SOS 采用哈希结构，提供 $O(1)$ 查找并消除索引更新的写放大，代价是失去排序（见 Table 2 对比）。这对点查为主、无 range-scan 需求的在线服务是可接受的权衡。

**紧凑内存设计**：索引全量驻留内存以保证"至多一次磁盘 I/O"。每条索引条目仅 30 字节：
- **Key 表示**：96 位 [[MD5]] 哈希（而非完整 key 字符串），空间高效且碰撞概率可忽略。
- **Timestamp**：48 位毫秒精度 Unix 时间戳，用于版本控制、冲突解决和 CAS 事务。
- **TTL 编码**：16 位变精度字段，支持最长 20 年、秒级粒度。
- **Service ID**：16 位标识租户，用于 SCAN 时的高效过滤。

**碰撞概率分析**（公式 3）：基于 [[Birthday Paradox|生日悖论]]，对 $H=2^{42}$（超 4 万亿）索引条目在 $2^{96}$ 表示空间下，碰撞概率约 $2^{-13}\approx 0.0122\%$，可忽略；磁盘存储模块还会在最终读取时验证完整 key 作为安全兜底。

**索引布局**：用[[分离链表|separate chaining]]解决哈希碰撞。索引由 32-bit head index 的 bucket 数组 + 预分配的 30 字节条目池组成，用数组下标而非机器指针使结构位置无关，可驻留共享内存。典型节点（256GB RAM，六块 4TB SSD）在 71% 磁盘利用率下，约 138GB 索引 + 少量写缓存，剩余约 100GB 用于读缓存，体现内存/磁盘资源的均衡利用。

**索引持久化**：通过 mmap 映射到磁盘并周期性 flush（OS page cache 或自定义实现）。即使索引 flush 领先于数据 flush，未落盘的数据块仍可从写缓存读取（写缓存总是先于磁盘被探测），因此读正确性不受磁盘持久化进度影响。

**TTL 支持**：后台例程周期清理过期数据；索引条目上的过期时间戳使 TEST/GET 可在不访问磁盘的前提下快速过滤过期数据。

### 核心模块 3：Primary-Based Chain Replication with Two-Phase Commit

SOS 在经典[[Chain Replication|链复制]]基础上做了三项关键修改：

1. **Head-based routing**：每个 key 由 1 个 primary + $R-1$ 个 secondary 副本服务，所有客户端读写都经过 primary（head），客户端只能观测到跨全部副本提交完成的数据。
2. **两阶段提交**：
   - **Phase 1（Reserve）**：head 生成创建时间戳；每个副本校验路由表版本、时间戳、TTL、CAS 条件，持久化数据到磁盘，预留索引槽位但不更新索引；该阶段从 head 向 tail 正向传播；任一副本校验或持久化失败则沿链反向回滚，不留下部分提交状态。
   - **Phase 2（Commit）**：tail 发起提交，更新自己的索引使数据可见；commit 消息反向传播回 head，每个副本在收到成功通知后更新索引。
3. **读/删除语义**：读查询 head，只返回索引可见（即成功提交）的数据；删除遵循同样的两阶段协议。

**关键优势**：
- **可靠性**：把失败率高的操作（校验、磁盘写）解耦进 Phase 1，轻量操作（索引更新）放进 Phase 2，降低 Phase 2 失败概率；Phase 1 失败触发原地回滚，罕见的 Phase 2 失败通过持续重试直至下游恢复或被 ConfigServer 心跳超时永久剔除后在新链上继续提交。
- **线性一致性**（[[Linearizability]]）：通过三个机制保证——未提交版本对读严格不可见；[[Lock Striping|锁条带]]（lock striping）序列化 Phase 2 索引更新定义线性化点；路由表变更时 ConfigServer 向所有 DataServer 广播新版本，携带旧版本的在途请求被拒绝并提示表刷新，传播窗口被限制在一个 RTT 内（小于同一客户端因果相关操作间的最小间隔）。
- **操作简洁性**：所有读经 primary，客户端无需追踪 tail 身份；节点对称角色简化了重试逻辑。

**网络优化**：DataServer 间通信使用 C++ [[Workflow]] 异步框架，将网络 I/O 与计算分离，充分并发利用 CPU、网络、磁盘资源，避免软件层调度成为瓶颈。

### 核心模块 4：Compare-And-Swap (CAS) Transactions

CAS 直接构建于两阶段提交协议之上。每个请求携带 `compare_timestamp`：
- PUT/DELETE：仅当现有版本时间戳 $\le$ `compare_timestamp` 时才允许覆写。
- GET：仅当存储版本时间戳严格大于 `compare_timestamp` 时才返回数据。

**分布式环境中的原子性保证**：网络重排序意味着到达某一副本的顺序不代表到达下一副本的顺序一致，SOS 通过两阶段提交保证原子性：Phase 1 沿链顺序比较 `compare_timestamp` 与本地索引时间戳，Phase 2 由于索引更新经锁条带序列化，并发的相同 `compare_timestamp` 事务只有一个能成功提交（见 Figure 5 的 Transaction A/B 示例）。失败时沿原链回滚，释放未提交资源。

**三种典型用例**：
1. **原子读改写**：客户端把读到的时间戳作为 `compare_timestamp`，防止并发写导致的更新丢失。
2. **不存在才插入**：设 `compare_timestamp=1`，确保已有数据永不被覆写。
3. **条件检索**：客户端用最后见到的时间戳抑制未变数据的传输，减少带宽。

毫秒精度时间戳在因果依赖请求之间（受制于数据中心内 RTT > 1ms）已足以保证版本排序，即便在 CAS 事务中也足够。

### 核心模块 5：Routing and Replication

数据分片与路由由 ConfigServer 管理，职责是维护路由表与通过心跳监控 DataServer 存活。与[[Consistent Hashing|一致性哈希]]（牺牲全局排序换负载均衡）不同，SOS 采用**固定 segment 分区**，实现确定性的 $O(1)$ 表查找：数据切分为 $S$ 个 segment（如 $S=16{,}384$），每个 segment $R$ 副本，共 $S \times R$ 个数据副本分布在 $N$ 个 DataServer 上。

**路由调度约束**（线性规划问题）：
1. Primary 副本数在各节点均衡（读负载均衡）；
2. 总副本数在各节点均衡（存储/写负载均衡）；
3. 同一 segment 的各副本避免落在同一节点（容错）；
4. 若配置了逻辑机架（logical rack / `group_id`），同一 segment 的各副本优先避免落在同一逻辑机架（地理容错）。

优化目标是最小化节点变化时需要调整的 segment 数，加快数据迁移/复制与集群收敛。

**逻辑机架**：通过 `group_id` 概念增强容错，分配算法尝试让单个 segment 的各副本互斥于其 `group_id`。对 $R$ 副本集群配置 $R$ 个不同逻辑组，可保证每个逻辑区域都有一份完整数据拷贝,支持低延迟本地读与无缝跨区域容灾（同城跨机房 RTT 约 2ms，3 副本链操作约额外增加 4ms 写延迟）。

**ConfigServer 可用性**：由于客户端本地缓存路由表、只在初始化或版本不匹配时联系 ConfigServer，CS 不是性能瓶颈;路由表持久化并复制到 standby，故障时通过 DNS 更新提升 standby，仅影响正在主动拉取新表的客户端。

### 可用性与运行时管理（§3）

SOS 在三个时间尺度上处理节点故障：

- **In-chain Retry（瞬时不可用，§3.1）**：请求发起者（client 或上游 DataServer）检测到不可用节点时，利用路由表切换到同一 shard 的另一副本重试，防止整个链操作失败。写/删除：若请求含两个以上地址，把不可用地址放回包并试下一地址；若只剩一个地址（即下一节点是链尾），把当前请求放入内部重试模块并立即向上游返回成功，重试模块持续尝试直到该节点恢复或被永久剔除。读/测试：目标主要是单个节点，head 失败则尝试另一持有数据的副本，仅在全部 $R$ 副本都失败后才算失败。Table 5 量化：写删除可容忍 1 个瞬时不可用副本（容错率 $1/R$），只读请求 $R$ 副本中 1 个可用即可完成（容错率 $(R-1)/R$）。生产中为推荐系统部署 20ms 超时重试 SDK 后，硬件/网络抖动期间的失败率从 4% 降到 0%。
- **Failure Detection and Health Checking（永久剔除，§3.2）**：DataServer 内部状态机（Initializing / Running / Unhealthy）自检本地故障（如单盘 I/O 错误累计超阈值），ConfigServer 通过心跳机制检测跨集群的不健康节点（心跳超时视为失败），实现 [[Failure Detector|Chandra & Toueg 意义上]]的"最终完美"故障检测器。剔除流程：移除路由表 → 为受影响 segment 提升 surviving follower 为 primary（若会消灭某 segment 的全部副本则拒绝剔除）→ 路由表版本号递增并广播 → 客户端请求过期版本号被拒绝后刷新路由表。
- **Data Re-replication and Cluster Balancing（冗余恢复，§3.3）**：集群四种状态——Uninitialized、Fully Replicated、Under-replicated、Migration/Replication in Progress（除 Uninitialized 外均可正常服务）。所有状态转移手动触发，除心跳超时导致的自动剔除。ConfigServer 校验操作合法性（当前集群状态是否允许、新旧节点心跳是否稳定、无不可用的 retry 发起节点），再按 §2.6 约束更新路由表并下发迁移命令。DataServer 以 Segment 粒度迁移：**Scale-out（扩容）**——数据迁移完成后源节点立即删除本地拷贝以释放空间，迁移期间源节点把该 segment 请求转发给目标节点（带当前时间戳避免被旧时间戳数据覆盖）；**Replica Repair（修复）**——源节点复制数据恢复缺失副本但保留自己的拷贝,转发逻辑同 Scale-out。全部节点上报成功后，ConfigServer 广播新路由表版本。

---

## 关键公式

### 公式 1：[[Step Sequence|Block 大小的 Step 序列]]

$$
S_i = \text{block\_size} \times j \times 2^k, \quad j \in \{1, 3, 5, 7\},\ k \ge 0
$$

**含义**：定义 dstorage 中可用 block 规格的离散集合，相比纯 2 的幂次分配提供更细粒度，尤其利于小对象（多数分配发生区间）。

**符号说明**：
- $\text{block\_size}$：集群初始化时确定的基准块大小；
- $j \in \{1,3,5,7\}$：步进系数，在同一个 2 的幂次区间内插入 4 个档位；
- $k \ge 0$：指数级别，决定整体规格的量级。

### 公式 2：均匀分布数据下的期望浪费率

$$
E[W] = \frac{1}{N-1} \sum_{i=1}^{n} \frac{(S_i - S_{i-1})^2}{2S_i},
$$

**含义**：量化 step-sequence 分配方案在数据大小均匀分布于 $[1, N]$ 时的平均内部碎片率；对 $N = 64\text{MB}$，该方案达到约 7.93% 的期望浪费率，远低于传统 2 的幂次分配的 25%–30%。

**符号说明**：
- $S_0 = 0$，$S_i$ 为公式 1 定义的第 $i$ 档 block 规格；
- $n$：覆盖 $[1, N]$ 所需的档位数；
- $N$：数据大小上界（如 64MB）。

### 公式 3：96-bit MD5 索引的碰撞概率

$$
P_{\text{coll}} \approx 1 - e^{-\frac{H^2}{2M}} = 1 - e^{-\frac{2^{84}}{2^{97}}} = 1 - e^{-2^{-13}} \approx 2^{-13}
$$

**含义**：基于[[Birthday Paradox|生日悖论]]估算在给定索引条目规模下，用 96-bit MD5 哈希代替完整 key 导致误碰撞的概率，验证压缩 key 表示在实际集群规模下是安全的。

**符号说明**：
- $H$：单个节点（因 SOS 路由机制同样基于 key 的 MD5 哈希，碰撞 key 必落同一节点）承载的索引条目数，论文取 $H = 2^{42}$（超 4 万亿，对应至少 1024 个 DataServer）；
- $M = 2^{96}$：96-bit MD5 的表示空间大小；
- 结果 $2^{-13} \approx 0.0122\%$，可忽略，且磁盘存储模块会用完整 key 做最终安全校验。

---

## 关键图表

> SOS PDF 页面已以高分辨率 PNG 渲染至 `assets/SOS/`，完整保留论文全部 14 个 Figure 和 5 个 Table 的原始图表、坐标与 caption。

### Figure 1：SOS 架构总览（论文 p1）

![[assets/SOS/SOS_page01.png]]

**说明**：SDK/Proxy 作为客户端入口，连接 ConfigServer（管理路由表）与多个 DataServer；每个 DataServer 内部包含 Hash Index Table、Read LRUCache 以及多个 dstorage + write cache 单元，DataServer 之间互相通信以完成链复制。

### Figure 2：SOS 系统能力与业务负载的协同演进（2013–2025，论文 p2）

![[assets/SOS/SOS_page02.png]]

**说明**：系统能力演进轴（Distributed Cluster → Chain Replication → Push/Pop Pipeline → Async Network/SSD Optimization → CAS Transaction → Custom Block & Eviction Policy）与业务负载轴（输入法/浏览器 → 搜索引擎缓存 → 推荐 → LLM/RAG/KV-cache）平行展示 SOS 从单机 KV 存储成长为 PB 级分布式基础设施的历史轨迹。

### Figure 3：请求处理流水线与存储层级（论文 p3）

![[assets/SOS/SOS_page03.png]]

**说明**：读路径（步骤 ①–⑥）经 Hash Index → Read Cache（LRU+LFU）→ Disk Storage，多级缓存与哈希索引共同保证低延迟；写路径（步骤 ①–⑤）经 Write Cache 立即返回响应（④），再异步 flush 到 SSD/HDD。图中区分同步数据流（实线）与异步数据流（虚线），并标出 Lock Striping 在索引更新上的作用。

### Figure 4：磁盘物理存储布局（论文 p3）

![[assets/SOS/SOS_page03.png]]

**说明**：Disk Storage 按 Step Info 划分 step-0/step-1/step-2/…，每个 step 对应一组同规格的 chunk（chunk-0…chunk-7）；放大的 chunk-7 展示其 Header（含 block_size=4096）与内部顺序排列的 block-0…block-1023，每个 block 存储 id、key、value 等字段。

### Figure 5：并发 CAS 事务时间线（论文 p5）

![[assets/SOS/SOS_page05.png]]

**说明**：Transaction A（绿）与 Transaction B（蓝）尝试用相同 `cas_timestamp = t1` 更新同一 key。通过 Primary 与 Follower 间的两阶段提交，系统强制线性一致性：Transaction B（$t_3$）先提交成功，Transaction A 因冲突检测（$t_3 > t_1$）被拒绝 `cas_error`，Transaction A 回滚并释放预留的 block，无可见副作用。

### Figure 6：链重试场景（论文 p7–8）

![[assets/SOS/SOS_page08.png]]

**说明**：四种场景——(1) Tail Failure：尾节点失败后重试成功；(2) Intermediate Failure：中间节点失败，重试绕过；(3) Primary Failure：primary 失败，客户端重组链、换新 head 重发；(4) Multiple Failures：多节点同时失败导致响应错误。场景 1–3 展示背景重试（④a/⑤a）与提交（④b/⑤b）如何并行推进。

### Figure 7：集群状态转移图（论文 p9）

![[assets/SOS/SOS_page09.png]]

**说明**：集群在 Uninitialized → Fully Replicated ↔ Under Replicated ↔ Replication/Migration in Progress 四态间转移；矩形代表手动命令（allocate / balance / fix / stop / finish），菱形代表自动系统条件（heartbeat timeout / clear unhealthy）。

### Figure 8：吞吐 vs. 客户端线程数（Zipfan/Uniform，论文 p10）

![[assets/SOS/SOS_page10.png]]

**说明**：(a) Zipfan 分布下 SOS 在 YCSB-A/B/C 上随线程数（16→256）近线性扩展，256 线程峰值达 1.86M QPS；(b) Uniform 分布下吞吐整体更低但差距模式相似。实线为 SOS，虚线为 RocksDB，RocksDB 在两种分布下均显著落后。

### Figure 9：P99/P99.9 延迟 vs. 客户端线程数（Zipfan，256 线程，论文 p10）

![[assets/SOS/SOS_page10.png]]

**说明**：左侧 SOS（线性坐标）P99/P99.9 延迟随线程数温和上升（P99 最高约 2.0ms，P99.9 最高约 6–10ms）；右侧 RocksDB（更大坐标尺度）同条件下 P99 高达 35.9ms、P99.9 高达 50.5ms，凸显 SOS 在尾延迟上的巨大优势。

### Figure 10：64 线程下吞吐与 P99（Zipfan 上 / Uniform 下，论文 p10）

![[assets/SOS/SOS_page10.png]]

**说明**：Zipfan 下 SOS 相对 RocksDB 吞吐提升 6.8×（YCSB-A）/11.4×（YCSB-B）/10.0×（YCSB-C），P99 延迟降低 10.6×/10.6×/8.2×；Uniform 下吞吐提升 12.0×/8.8×/6.6×，P99 延迟降低 26.7×/9.6×/6.8×，YCSB-A（随机写）差距最大，印证 LSM-tree 在无局部性随机写下的 compaction 代价。

### Figure 11：写放大（WAF）与读放大（RAF）vs. 线程数（论文 p11）

![[assets/SOS/SOS_page11.png]]

**说明**：(a) SOS 全线程数下 WAF = 1.00×，RocksDB 为 14.5×–16.3×（因强制 LSM compaction）；(b) SOS RAF 恒为 1.00×（每次读恰好一次物理 I/O），RocksDB RAF 约 11.6×（因多层 bloom filter 探测与 block 读取）。实线为应用层 QPS（左轴），虚线为设备 IOPS（右轴）。

### Figure 12：缓存敏感性与不存在 key 的可扩展性（论文 p11）

![[assets/SOS/SOS_page11.png]]

**说明**：(a) YCSB-C 吞吐随缓存命中率变化（对数坐标）：All-hit（纯 DRAM）下 SOS/RocksDB 比值 25.5×，Zipfan（≈1% 命中）降为 10.0×，Uniform（≈0% 命中）降为 6.6×，表明 CPU 开销与 I/O 放大是独立贡献的两个因素；(b) 查询不存在 key（移除全部磁盘 I/O，隔离索引层本身）时两系统均达到约 29–30M QPS（64 线程，1.047× 差距），说明索引层本身对负查找并无显著优势，吞吐提升主要来自 I/O 放大与 compaction 开销的消除。

### Figure 13：Value 大小敏感性（100% 写，Uniform，64 线程，论文 p11）

![[assets/SOS/SOS_page11.png]]

**说明**：Value 大小从 4KB 增至 16KB，吞吐从 246k 降至 152k QPS，P50/P99 延迟相应平滑上升（P99 从 211μs 升至 2.62ms），证明 step-sequence block 布局不引入额外结构开销，延迟/吞吐变化完全由更大 I/O 传输量本身解释，"一次 I/O"保证在全部测试对象大小范围内保持成立。

### Figure 14：Portrait South 生产集群 48 小时监控轨迹（2026年2月25–26日，论文 p12）

![[assets/SOS/SOS_page12.png]]

**说明**：单实例 48 小时真实流量轨迹。(a) QPS 峰值：GET≈50k、PUT≈80k；(b) P99.9 延迟：PUT < 8ms、GET < 500μs；(c) CPU 占用 < 2,200 millicores。验证单节点基准测试结果与真实生产负载下的表现一致。

### Table 1：SOS 支持的命令集（论文 p2）

| Command | Description | Replicas Involved |
|---|---|---|
| PUT | Write/Update a KV pair | R |
| DELETE | Delete a KV pair | R |
| GET | Retrieve a value by Key | 1 |
| TEST | Check for Key existence | 1 |
| MGET | Retrieve multiple values locally | 1 |
| PUSH | Batch prefetch value by sequential scan | 1 |
| POP | Retrieve prefetched batch KV pairs | 1 |
| SCAN | Retrieve all Keys or KV pairs | 1 |

**说明**：PUT/DELETE 需同步全部 $R$ 个副本，GET/TEST/MGET/PUSH/POP/SCAN 均为单副本操作；PUSH/POP 组合支持跨 DataServer 的大规模批量顺序扫描检索。

### Table 2：索引结构对比（Hash / LSM-tree / B+ Tree，论文 p4）

| Property | Hash | LSM-tree | B+ tree |
|---|---|---|---|
| Point query | $O(1)$ | $O(L\log n)$ | $O(\log n)$ |
| Range query | N/A | $O(k + L\log n)$ | $O(k + \log n)$ |
| Write amplification | 1× | 10–30× | 1–2× |
| Disk I/O per read | 1 | $[1, L]$ | $\lceil \log_B n \rceil$ |
| Space Efficiency | High | Medium | Medium/High |

**说明**：$L$ 为 LSM-tree 层数，$k$ 为 range query 返回条目数，$B$ 为 B+ 树分支因子（磁盘系统典型值 100–200）。冷缓存点查在 LSM-tree 上需读取多达 $L$ 个数据块加索引/过滤块，生产中有效读放大为 2–12×。该表为 SOS 选择哈希索引提供了直接依据。

### Table 3：Segment 路由表示例（4 Segment、3 副本、6 DataServer，论文 p6）

| Segment | Replica 0 (Primary) | Replica 1 | Replica 2 |
|---|---|---|---|
| 0 | DS1 | DS2 | DS3 |
| 1 | DS4 | DS5 | DS6 |
| 2 | DS2 | DS3 | DS4 |
| 3 | DS5 | DS6 | DS1 |

**说明**：固定 segment 分区机制的最小示例，展示 primary/follower 如何在节点间均衡分布，保证 $O(1)$ 路由查表的同时满足容错约束（同一 segment 的副本不落在同一节点）。

### Table 4：引入逻辑机架约束后的路由表（论文 p7）

| Segment | Replica 0 (Primary) | Replica 1 | Replica 2 |
|---|---|---|---|
| 0 | DS1 | **DS5** | DS3 |
| 1 | DS4 | **DS2** | DS6 |
| 2 | DS2 | **DS6** | DS4 |
| 3 | DS5 | **DS3** | DS1 |

**说明**：假设三个逻辑组 group1={DS1,DS2}、group2={DS3,DS4}、group3={DS5,DS6}，相对 Table 3，加粗项为调整后的副本分配，使同一 segment 的所有副本互斥于逻辑组，达到最小化副本迁移次数的下界。

### Table 5：按命令类型的重试行为（论文 p8）

| Property | PUT / DEL | GET / TEST |
|---|---|---|
| DataServer Per Attempt | $R$ | 1 |
| Max Attempts | $\max(1, R-1)$ | $R$ |
| Fault Tolerance | $1/R$ | $(R-1)/R$ |

**说明**：Max Attempts 为总尝试次数（含首次+重试）；$R=1$ 时 PUT/DEL 恰好一次尝试。写删除可容忍 1 个瞬时不可用副本，只读请求容忍 $(R-1)/R$ 比例的副本不可用。

---

## 实验

### 实验环境

| 项目 | 内容 |
|---|---|
| 硬件 | Intel Xeon Platinum 8374C（132 核）、382GB DRAM、Samsung PM9A3 NVMe SSD（3.84TB，4KB native LBA），Linux I/O scheduler 设为 none |
| 对比基线 | RocksDB v7.0（同配 4GB BlockCache、启用 Bloom filter、关闭压缩、max_background_jobs=8） |
| SOS 配置 | Read Cache 限制为 4GB（≈数据集 1%），启用 DirectIO；Write Cache Ratio=25%；两个 dstorage 实例各配 2 个背景 flush 线程（共 4 个） |
| 负载 | YCSB，1 亿 key（20 字节 key，4000 字节 value），约 374GB 原始数据集；标准 profile：YCSB-A（50% 写）、YCSB-B（5% 写）、YCSB-C（只读） |

论文说明只做单节点存储引擎对比：分布式 KV 系统引入副本协议、网络拓扑、集群调优等混淆变量，难以在共享测试台上做受控比较；排除 [[Redis]] 因其单线程模型使线程数对比在结构上不公平，且 SOS 保证至多一次磁盘 I/O，任何差距主要反映网络框架差异而非存储效率。

### 关键实验结论

1. **吞吐**（§4.2）：Zipfan 读多负载下 SOS 近线性扩展，256 线程峰值 1.86M QPS；64 线程下 YCSB-B/C 分别是 RocksDB 的 11.4×/10.0×；写多负载（YCSB-A）因 4 个背景 flush 线程饱和在 390–470k QPS，仍是 RocksDB 的 6.5–6.8×。
2. **延迟**（§4.2）：Zipfan YCSB-A 64 线程下，SOS 处理 6.8× 更多请求但 P99 为 1.2ms（RocksDB 12.4ms，10.6×）；YCSB-C 下 SOS P99 低于 172μs（仅约 1% 缓存覆盖率）；128 线程下 YCSB-A 的 SOS P99.9 为 5.4ms（在 10ms 生产 SLA 内），RocksDB 为 51.9ms（已超出）。
3. **负载敏感性**（§4.2）：Uniform 分布下 4GB 读缓存实际冷命中（≈0%），暴露原始 I/O 效率。YCSB-A（随机写）差距最大：SOS 392kQPS @ P99=1.1ms vs. RocksDB 33kQPS @ P99=30.6ms（吞吐差 12.0×，P99 差 26.7×）；只读 YCSB-C 下 SOS 554kQPS，仍是 RocksDB 最佳 Zipfan 结果（175kQPS）的 3.2×，证明性能差距不依赖缓存局部性。
4. **写/读放大**（§4.3）：SOS 全线程数下 WAF=RAF=1.00×；RocksDB WAF 14–16×、RAF ≈11.6×。
5. **组件消融**（§4.4）：缓存命中率敏感性实验表明 CPU 开销（25.5×，全命中）与 I/O 放大（降至 6.6×，Uniform 冷缓存）贡献独立；负 key 查找（隔离索引层）两系统均达 ≈29–30M QPS（1.047× 差距），说明吞吐优势来自 I/O 放大消除而非索引结构本身；value size 从 4KB 到 16KB 的吞吐/延迟平滑退化，验证 step-sequence 布局无结构性开销。
6. **可扩展性分析**（§4.5）：垂直扩展——$O(1)$ 哈希查找 + 单 I/O 保证使单节点吞吐随线程数线性增长直至设备带宽耗尽；水平扩展——路由表提供与集群规模无关的 $O(1)$ 查表，扩容只需路由表版本号递增与定向 segment 迁移，无需全局重索引；副本扩展——写成本随副本数 $R$ 线性增长 $O(R)$（每写需提交全部副本），读延迟不受影响（只查 primary）。

### 生产经验（§5）

- **规模**：腾讯 40+ 生产集群，总部署超 50PB，覆盖元宝（AI 搜索与 RAG）、混元（LLM 推理）、腾讯视频、腾讯新闻、QQ 浏览器、搜索引擎与推荐系统。
- **Portrait South Cluster**（192 实例，3 副本）：相对遗留存储系统，客户端平均读延迟从 1.7ms 降至 0.7ms（降 60%），P99.9 稳定在 5.0ms；日数据摄取流水线端到端处理时间从 20 小时降至 9 小时，保持 100% 成功率。
- **Yuanbao South Cluster**（912 实例，2 副本）：存储 1.66 万亿 key（$1{,}664.9\times10^9$），容量 15.9PB，数据集仍快速增长，验证了水平扩展能力——仅需路由表更新即可添加 DataServer，数据迁移无服务中断。
- **Yuanbao East Cluster**（320 实例，3 副本）：持续峰值 SCAN 吞吐 62M QPS，验证有状态 SCAN 路径在生产规模下的表现。
- **成本效益**：相对 Redis 迁移到 SOS 的生产服务，内部计费数据显示平均 TCO 降低 84%。

---

## 批判性思考

### 优点

1. **设计不变式清晰且贯穿全文**："至多一次物理磁盘 I/O"这一单一不变式作为设计锚点，索引、磁盘布局、写缓存三个子系统的设计选择都能回溯到这一目标，论证逻辑一致、可解释性强。
2. **实验设计有针对性地隔离变量**：组件消融实验（缓存敏感性、负 key 查找、value size 敏感性）明确区分了"I/O 放大消除"与"纯 CPU/索引层开销"两类收益来源，避免把全部性能提升笼统归因于某个单一设计点，这一点在系统论文中并不常见、值得称道。
3. **十余年真实生产数据支撑**：从 2013 年至今的持续演进历史、40+ 生产集群、50PB+/7 万亿记录规模、48 小时真实流量轨迹，提供了罕见的长周期工业验证，比单纯的 benchmark 结果更有说服力。
4. **工程取舍务实**：写缓存引入的"节点级一致性风险换吞吐"的权衡被清晰地论证（多副本协议兜底、年度生产无数据丢失记录），而非掩盖这一代价。

### 局限性

1. **牺牲 range query 能力是硬限制**：哈希索引的选择决定了 SOS 从根本上不支持范围查询，论文在 Limitations 中坦承这点，RAG 场景仍需额外的向量检索引擎做候选召回，SOS 本身只是精确 key-value 层。
2. **分布式实验证据相对单薄**：§4 评估聚焦单节点存储引擎对比（论文明确说明是为了排除混淆变量），分布式链复制协议本身的吞吐/延迟/故障恢复表现主要靠生产集群的描述性数据（Figure 14、§5 案例）支撑，缺乏受控的分布式 benchmark（如不同 $R$、不同故障注入模式下的系统性对比）。
3. **写缓存的一致性风险依赖"软性兜底"**：进程异常退出或机器宕机时未 flush 数据的处理方式是把节点逐出路由表、依赖多副本协议恢复，这本质上接受了短暂的降级窗口（under-replicated 状态），对强一致性要求极高或副本数较低（如 $R=2$）的场景风险更高，论文未充分讨论 $R=2$ 下双节点同时故障的处理。
4. **CAS 粒度为单 key，不支持多 key 事务**：论文的 CAS 设计服务于读改写、插入去重、条件检索三类单 key 场景，没有涉及跨 key 的原子性保证，对需要多 key 事务语义的应用（而非单纯 KV 查找）不适用。
5. **碰撞概率分析的前提依赖集群规模假设**：96-bit MD5 碰撞概率分析中 "$H=2^{42}$ 意味着至少 1024 个 DataServer" 的论证链条依赖"碰撞 key 必落同一节点"这一假设（因路由也基于 MD5），若未来路由策略变化，该安全性论证需要重新审视（不过论文已指出磁盘层会做完整 key 校验作为兜底）。

### 潜在改进方向

1. 补充分布式环境下的受控故障注入实验（不同 $R$、不同故障模式、不同集群规模下的恢复时间与可用性曲线），弥补单节点实验与生产案例描述之间的证据空隙。
2. 探索多 key 原子操作（如同机关联 key 的批量 CAS）支持，扩大 CAS 机制的适用场景。
3. 针对 LLM KV-cache workload 的 block_size 自适应（当前依赖模型架构手动设置 block_size 以降低碎片）可考虑运行时自动探测 tensor 规格分布并动态调整 step 序列。
4. 结合 LLM 调度器更紧耦合的集成方案（论文自身列为未来工作）：当前领域特定逻辑（多轮 KV-cache 上下文映射、复合结构编码）需应用层自行处理，若能提供原生复合类型支持可降低上层集成成本。

### 可复现性评估

- [x] 代码开源（https://github.com/holmes1412/sos-vldb2026-supplementary）
- [ ] 预训练模型（不适用，系统论文）
- [x] 实验配置细节（硬件规格、SOS/RocksDB 对比配置）较为完整
- [x] 公开数据集（YCSB）可获取；生产集群数据不可获取

---

## 关联笔记

### 基于

- [[Chain Replication]]：SOS 的 Primary-Based Chain Replication 直接扩展自经典链复制协议，修改读路径（从 tail 改为 head）并引入两阶段提交。
- [[Two-Phase Commit]]：SOS 用两阶段提交实现链上的强一致写入，但把失败率高的操作放进 Phase 1 以降低 Phase 2 失败概率，这一设计相对经典 2PC 做了针对性优化。
- [[Linearizability]]：SOS 通过 head-based 读路由 + lock striping 序列化索引更新实现线性一致性。
- [[Failure Detector|Chandra-Toueg Failure Detector]]：ConfigServer 的心跳健康检查实现了 Chandra & Toueg 意义上的最终完美故障检测器。

### 对比

- [[RocksDB]]：论文的核心基线，代表 [[LSM-tree]] 式磁盘 KV 存储标准实现；SOS 在 WAF、RAF、吞吐、尾延迟上全面超越，核心原因是消除了 compaction 带来的写放大与多层查找带来的读放大。
- [[Redis]]：内存 KV 存储代表，因 PB 级下成本过高（约 10×–30× 于 NVMe 方案）而被排除在实验对比之外,但在相关工作与生产 TCO 对比中作为参照。
- [[FASTER]]：共享"至多一次 I/O"设计理念（hybrid log），但针对单机吞吐优化而非分布式强一致性,是设计理念上最接近的学术系统。
- [[Anna]]：通过基于 lattice 的无协调设计实现极致读扩展性，但面向最终一致性，缺少 SOS 所需的强一致性与 TTL 语义。
- [[CRAQ]]：经典链复制的另一变体，通过多副本访问扩展读吞吐但引入脏读风险（需查 tail 版本），与 SOS 的 head-based 单跳读路径设计方向不同。
- [[DeepSeek 3FS]]：面向训练场景的大规模顺序带宽系统（6.6TiB/s），采用 CRAQ 复制；SOS 则针对 AI serving 场景的单请求尾延迟优化（P99.9 < 5ms）。

### 方法相关

- [[LSM-tree]]：论文反复以 LSM-tree 的写放大（10–30×）、读放大（最多 $L$ 层）作为对照，论证哈希索引 + 原地覆写方案的优势。
- [[Bitcask]]、[[SILT]]：同样使用哈希索引实现 $O(1)$ 查找的系统，但依赖周期性 merge 回收 append-only log 空间；SOS 用原地覆写消除此开销。
- [[Consistent Hashing]]：SOS 显式对比并放弃一致性哈希（为负载均衡牺牲全局排序），转而采用固定 segment 分区实现确定性 $O(1)$ 路由。
- [[MD5]]：SOS 索引用 96-bit MD5 哈希压缩 key 表示，兼顾空间效率与可忽略的碰撞概率。
- [[Birthday Paradox|生日悖论]]：用于分析索引 key 碰撞概率的核心概率论工具。
- [[KV Cache]]、[[RAG]]：SOS 服务的两类重要 AI 场景负载，驱动了其对低尾延迟、点查语义的设计选择。
- [[Workflow|C++ Workflow]]：SOS 的底层异步网络通信框架，解耦网络 I/O 与计算。
- [[Direct IO]]：SOS 磁盘层绕过 OS page cache、自行实现写回管理的底层 I/O 机制。

### 硬件/数据相关

- [[YCSB]]：论文使用的标准 KV 存储 benchmark，覆盖 YCSB-A/B/C 三种典型负载模式。
- [[NVMe SSD]]：SOS 面向的核心存储介质，相对内存实现约 10× 成本降低。

---

## 速查卡片

> [!summary] SOS
> - **核心**：保证每次 KV 操作至多一次物理磁盘 I/O，用哈希索引 + 分级变长磁盘布局 + 异步写缓存三件套在 SSD 成本上逼近内存存储延迟。
> - **方法**：$O(1)$ 内存哈希索引（96-bit MD5 key）+ step-sequence block/chunk 磁盘引擎（碎片率 ≈7.93%）+ 异步 flush 写缓存 + Primary-Based Chain Replication with Two-Phase Commit + in-chain retry。
> - **结果**：WAF=RAF=1.00×（vs. RocksDB 14–16×/11.6×）；704K QPS @ P99=2ms（YCSB-B，256 线程）；相对 RocksDB 吞吐最高 12.0×、P99 最高 26.7× 改善；生产部署超 50PB/7 万亿记录，峰值 550M QPS，P99.9 读延迟 <5ms，TCO 相对内存存储降 84%。
> - **代码**：开源于 https://github.com/holmes1412/sos-vldb2026-supplementary

---

*笔记创建时间: 2026-10-10*
