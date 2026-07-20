---
title: "Twenty Years of Bigtable"
method_name: "Bigtable"
authors: [Fabio Baltieri, Bora Beran, Igor Bernstein, Sailesh Krishnamurthy, Jeremy Hurwitz, et al.]
year: 2026
venue: SIGMOD Companion
tags: [bigtable, wide-column-store, lsm-tree, distributed-storage, replication, google, non-relational]
zotero_collection: SIGMOD/2026
image_source: local
created: 2026-07-09
---

# 论文笔记：Twenty Years of Bigtable

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | Google |
| 日期 | May 2026 |
| 项目主页 | https://cloud.google.com/bigtable |
| 对比基线 | [[DynamoDB]], [[HBase]], [[Cassandra]] |
| 链接 | [ACM DL](https://dl.acm.org/doi/abs/10.1145/3788853.3803095) |

---

## 一句话总结

> Google 回顾 Bigtable 二十年的演进历程，从最初的简单架构发展至管理 10EB 数据、70 亿 QPS 的超大规模存储系统，通过复制、SQL 支持、CRDT、物化视图等新特性以及持续的性能/可靠性/资源效率优化，证明了可扩展架构的持久生命力。

---

## 核心贡献

1. **复制系统**: 介绍了 Bigtable 自原论文以来最重要的新功能 —— 多主复制（multi-primary replication），支持最终一致性、拉取式复制模型、复制水位线跟踪，以及 BEAR 机制实现无停机添加新副本
2. **库内处理**: 新增 SQL 查询接口（GoogleSQL 扩展）、[[Change Data Capture|CDC]]、[[CRDT]] 计数器/聚合类型、原生 [[Materialized Views|物化视图]]，将计算推送到数据库引擎内部
3. **可扩展性与性能优化**: 外部压缩（external compaction）、行级缓存、分片 [[Bloom Filter]]、自动扩缩容等多维度改进，支撑了 10EB 数据规模

---

## 问题背景

### 要解决的问题

Google 在 2004 年左右构建了 Bigtable 作为非关系型数据库系统，最初发表于 OSDI 2006。二十年来，随着数据量和用户需求的持续增长，Bigtable 需要在保持原有简单架构的基础上，不断添加新功能并改进性能、可靠性、可扩展性和资源效率。本文是对这二十年工程实践的系统性回顾。

### 现有方法的局限

- 原始 Bigtable 没有复制功能，无法满足全球可用性需求
- 缺少 SQL 接口，用户需要外部计算框架（MapReduce/Flume/Spark）进行数据分析
- 不支持计数器/聚合类型，用户只能用昂贵的 read-modify-write 事务或复杂的外部收集器
- 物化视图依赖外部流水线，架构复杂、运维成本高

### 本文的动机

Bigtable 的核心优势在于其简单架构、灵活存储格式和对性能的关注。作者认为这些原则可以扩展而不需要推翻重来，通过异步处理（非前台写路径）和灵活 schema 来容纳新功能，同时持续优化各个维度以应对不断增长的需求。

---

## 方法详解

### 系统架构

Bigtable 的整体架构与原论文基本一致，保持了可扩展的设计：

- **数据模型**: 三维有序映射 `(row_key, column_key, timestamp) → value:bytes`
- **行组织**: 按 row key 字典序排列，划分为 [[Tablet]]（tablet）作为托管/加载/移动/负载均衡的基本单位
- **列组织**: column key 的结构为 `column_family:column_name`，column family 进一步分组为 [[Locality Group|locality group]]，每个 locality group 在压缩时生成独立的 [[SSTable]]
- **存储引擎**: 结合 [[WAL]]（commit log）和 [[LSM-tree]]，所有变更先写 WAL 再确认
- **集群结构**: Master + 大量 [[Tablet Server|tablet server]]，tablet server 分组为 partition，进一步分为 subpartition
- **底层存储**: 数据存储在分布式文件系统 [[Colossus]] 上，tablet 移动不需要数据迁移
- **元数据**: 所有元数据存储在 Bigtable 自身（自引用），启动元数据存储在 [[Chubby]] 中

#### 数据持久化流程

Bigtable 使用三种主要压缩类型管理 [[LSM-tree]]：

1. **Minor Compaction**: memtable 满时写入 SSTable
2. **Merge Compaction**: 当 SSTable 栈过深时，合并顶部 K 个 SSTable 为一个
3. **Major Compaction**: 合并所有 SSTable 为一个，执行数据删除和垃圾回收

### 核心模块

#### 模块1: 复制系统

**设计动机**: 实现全球可用性和数据冗余，支持多主写入

**复制概念**:
- 复制在 column family 粒度上镜像数据，可在同一集群或跨数据中心
- 一组互为镜像的 replica 组成 replication configuration
- Replica 之间不协调写入确认，提供高可用性和低成本

**一致性模型** — [[Eventual Consistency|最终一致性]]:
- 给定 replica 的 replication watermark 中，某 source 在某 row 的 replication timepoint 不会倒退
- 不同 source 在同一 row 上的 mutation 全局排序（sequencer = replication timepoint + source ID 作为 tie-breaker）
- 冲突通过 **last-write-wins** 策略解决
- ACID 保证仅限于单行

**复制水位线（Replication Watermark）**:

见 [Figure 1](#figure-1-复制水位线replication-watermark)。每个 replica 维护一个 replication watermark，是一个从 source 到 interval map（user row range → replication timepoint）的映射。

**拉取式复制（Pull-based Replication）**:
- 目标 replica 主动从源 replica 拉取 mutation，而非源 push
- 优势：减少源 replica 开销，简化状态跟踪和崩溃恢复
- Mutation 存储在 replication log（系统 locality group），key 为 `(source, timepoint, row)` 的编码 tuple
- Puller 协议保证 **exactly-once** 交付
- 引入 heartbeat mutation（约每分钟）确保无用户写入时 watermark 仍能推进

**BEAR（Bigtable Efficient Add Replica）**:
- 挑战：添加新 replica 时需复制现有 SSTable，同时处理 in-flight mutation
- 解决方案：为复制的 SSTable 分配人为较低的 sequencer，确保 in-flight mutation 的高 sequencer 在冲突解决时胜出
- 实现零写停机的新副本启动

**自动垃圾回收与复制**:
- 复制表中 GC 可能与其他 replica 上的用户删除竞争
- 为避免分歧，replica 通过发出 delete mutation（而非直接删除）来执行 GC
- 3+ replica 场景：GC 只能计算已复制到所有其他 replica 的版本数量（age-based GC 不受此限制）

#### 模块2: 库内处理

**设计动机**: 将计算推送到数据库引擎，减少外部流水线复杂度

**SQL 查询支持（Queryability）**:
- 集成 [[GoogleSQL]] 作为主要查询接口
- 支持集合类型（map/set）和 temporal filter（`as_of` 算子）进行时间点查询
- 支持 Protobuf descriptor，将序列化对象视为 SQL 中的用户自定义类型
- Row key schema 将不透明 byte-string 映射为强类型 SQL 列，支持优化的范围扫描和谓词下推

**[[Change Data Capture|CDC]]**:
- 实时接收表中所有 mutation，用于实时分析、数据同步、欺诈检测
- CDC stream 在 tablet 边界分片，支持三种维度的位置跟踪：行、replica、逻辑时间
- Continuation token 允许消费者在 tablet 分裂/合并时管理状态交接
- CDC Beam Adapter 运行在 [[Google Cloud Dataflow|Cloud Dataflow]] 上，保证至少一次执行

**[[CRDT]] 计数器与聚合**:
- 支持可交换操作（增/减计数器）和不可交换操作（删除/清除计数器）
- 每层 [[LSM-tree]] 存储 changelog 结构：
  - **Resolved subtotal**: 不会与其他层 sequencer 顺序重叠的操作的已结算小计
  - **Unresolved operations**: 可能有重叠的操作，包括 "future" 最近更新和 "past" 迟到到达
- Merge compaction 时合并 changelog，修剪已 resolved 的操作
- 删除操作直接移除 memtable changelog 中 sequencer 在删除之前的操作

**[[Materialized Views|物化视图]]**:
- 用户定义 GoogleSQL 查询作为转换，Bigtable 在数据到达时持续执行
- 使用 [[Coprocessor]] 框架在 tablet server 上运行 server-side MapReduce 流水线
- 流程：更新源表 → 记录 logical clock → mapper coprocessor 处理 → hidden shuffle table → reducer coprocessor 聚合 → finalizer 编码到输出表
- 使用 watermark 跟踪各阶段处理进度，确保最终一致性

#### 模块3: 可扩展性改进

**水平扩展**:
- 表无大小限制，通过自动分裂增加 tablet
- 冷 tablet 可以更大，热 tablet 更小，实现最佳性能
- 过多 tablet 导致管理开销增加（内存、CPU、位置缓存、rebalancer 负载）

**垂直扩展**:
- 优化单个 tablet server 托管更多 tablet
- 只读 tablet 释放 commit log 引用，减少恢复时间和资源
- [[Discard Compaction]]：不读取数据直接删除低水位以下的 replication log SSTable
- 文件构建（file building）移出 tablet server，释放 CPU/RAM
- 更大服务器需要 commit log 分片以维持/提升写吞吐

**外部压缩**:
- 最耗时的文件构建在 tablet server 之外由独立 job 处理
- 适用于 merge 和 major compaction（minor 在 tablet server 内完成，因数据来自内存）
- 文件构建 job 独立自动扩缩容，可使用更便宜的资源

**Master 可扩展性**:
- 细粒度锁：缓解 tablet 数量级增长导致的锁竞争
- 并行 schema 更新：替代原来的串行更新

**并行文件 GC**:
- 原来由 master 串行执行的 mark-and-sweep GC 移至 tablet server 并行化
- Master 协调分派工作，tablet server 执行昂贵的 metadata 读取和文件删除

**Tablet 定位可扩展性**:
- 引入位置 hint（旧 tablet server 缓存最近卸载 tablet 的位置）
- Location cache proxy：独立扩缩容的元数据查询代理，分流定位流量

**Rebalancer**:
- 基于 CPU/RAM/RPC 队列长度的平滑指标识别负载不均
- 选择 "较冷" 的 tablet 卸载（而非最热的），避免热点 tablet 移动导致的定位查找风暴
- 可视化工具帮助识别热点 key

#### 模块4: 性能优化

**Row Cache**:
- 在 block cache 之上缓存行粒度数据
- 点读 CPU 使用降低最高 25%
- 稀疏表示：仅缓存查询访问的数据，与 block cache 共享内存池
- 缺失数据时回退到 block 读取并填充 row cache

**[[Bloom Filter]] 优化**:
- **分片 Bloom Filter**: 为 key range 子集维护独立 Bloom filter，存储为数据使用相同索引/缓存系统，DRAM 占用降低一个数量级
- **混合 Bloom Filter**: 支持 column family only 和 column family + column 两种过滤方式，利用率提升 4x
- **本地缓存**: 同一查询内多个 column family/column 重复访问同一 Bloom filter 时缓存
- **Bloom filter 索引缓存**: 自定义内存索引避免首次访问的 3-block 二分搜索开销
- **自适应性**: 定期评估 Bloom filter 有效性，无效时自动关闭

**请求优先级**:
- 用户可为重要流量标注调度优先级
- 支持混合事务/分析（[[HTAP]]）工作负载的事务操作与分析操作之间的隔离

#### 模块5: 可靠性保障

**快照与备份**:
- Copy-on-write 快照，创建成本极低
- 备份从快照开始，后台进程复制底层文件到备份服务

**端到端校验**:
- [[CRC32]] 校验覆盖 row key、column key、value、mutation、文件级
- 认证加密（authenticated encryption）同时提供机密性和完整性
- 客户端 → tablet server → 存储层 → 返回客户端的完整路径校验
- 检测到错误时：操作报错/客户端重试/怀疑硬件故障则 crash server

**持续检查**:
- 压缩时验证 SSTable 中的校验和，错误则 tablet 下线隔离
- 文件构建 job 在 SSTable "安装"前完整回读、解密、解压、逐条验证
- Commit log 缺口检测、内存元数据完整性检查、加密密钥校验

#### 模块6: 资源效率

**自动扩缩容（Autosizing + Autoscaling）**:
- **Autosizer**: 集群内监控进程，亚分钟频率，基于 CPU/RAM/tablet 可用性决定增减 server
- **Autoscaling**: Google 全局资源管理系统，从 PA 资源池提供/回收资源
- 两组件设计：快速反应（slack subpartition 池）+ 定制化指标（tablet 可用性）

**缓存调优（Cache Right-sizing）**:
- 基于 TCO 模型的反馈循环组件（cache sizer）
- 持续监控 RAM 成本 vs CPU/IO 缓存缺失成本
- 动态调整每个 server 的缓存大小限制

**分层存储、离线访问、批量导入**:
- 分层存储按数据年龄分离冷热文件到不同存储介质
- 离线访问：batch worker 直接从 Colossus 读取 SSTable，绕过 tablet server
- Cloud Bigtable Data Boost：无服务器按需计算，完全隔离于服务集群
- 批量导入：batch worker 直接构建 SSTable，完成后更新元数据

---

## 关键图表

### Figure 1: 复制水位线（Replication Watermark）

![[Bigtable_fig1_replication_watermark.png]]

**说明**: 展示 replica R3 的复制水位线。R3 的行范围被分为 3 个 tablet（[\0, a)、[a, e)、[e, inf)），每个 tablet 独立跟踪来自每个 source（R1、R2、R3）的 replication position。不同 source 在不同行范围上的 replication timepoint 可能不同步，例如 R1 的 [c, d) 范围可能比 [d, e) 范围更最新，因为各 tablet 独立传播变更。

### Figure 2: 删除操作导致的不一致

![[Bigtable_fig2_inconsistency_delete.png]]

**说明**: 展示将删除操作直接应用到运行总计时，两个 replica 之间产生不一致的示例。由于删除是非可交换操作，在各 replica 上按到达顺序应用会导致不同的结果。这激发了 Bigtable 使用 changelog 结构来正确处理删除操作。

### Figure 3: Changelog 结构

![[Bigtable_fig3_changelog_structure.png]]

**说明**: 展示单个 [[LSM-tree]] 层的 changelog 结构。示例中 HWM=10、LWM=20，包含：(1) sequencer 10-20 之间的 resolved subtotal = 10；(2) 三个 recent updates：+6@22, +5@23, +1@24；(3) 两个 late arrivals：+1@4, +3@5。这种结构使得 Bigtable 能在不牺牲一致性的情况下高效处理跨 replica 的可交换和不可交换操作。

### Figure 4: Changelog 合并操作

![[Bigtable_fig4_changelog_merge.png]]

**说明**: 展示在 merge compaction 期间执行的 changelog 合并操作。合并后的 changelog 将各层的 resolved subtotal 合入新的 resolved subtotal，修剪已 resolved 的操作条目，保留仍未 resolved 的操作。删除操作发生时，sequencer 在删除之前的操作被直接移除。

### Figure 5: 物化视图的 MapReduce 流水线

![[Bigtable_fig5_mapreduce_pipeline.png]]

**说明**: 展示 Bigtable 原生物化视图使用的 server-side MapReduce 风格流水线。流程为：源表更新 → 记录 logical clock 到 hidden column → mapper coprocessor 处理 → hidden shuffle table 中间聚合 → reducer coprocessor 聚合 → finalizer 编码结果到输出表。各阶段通过 watermark 跟踪处理进度，确保最终一致性。

---

## 关键技术指标

| 指标 | 数值 | 说明 |
|------|------|------|
| 数据总量 | 10 EB | Bigtable 管理的总数据量 |
| 峰值总 QPS | 70 亿 (7 billion) | 全系统峰值查询速率 |
| 单表最大行数 | 1.6 quadrillion ($10^{15}$) | 单个表中的最大行数 |
| 单表最大数据量 | 1 EB+ | 单个表的最大数据量 |
| 单集群峰值 QPS | 2.5 亿 (250 million) | 单个集群的峰值查询速率 |
| 最大复制用户 | 200+ replicas | 最大的复制配置中的 replica 数量 |
| 典型复制延迟 | < 1 分钟 | 大部分数据的复制延迟 |
| Bloom filter 利用率提升 | 4x | 混合 Bloom filter 引入后的提升 |
| Row cache CPU 降低 | 最高 25% | 点读操作的 CPU 使用降低 |
| 分片 Bloom filter DRAM 节省 | 10x | DRAM 占用降低一个数量级 |

---

## 批判性思考

### 优点

1. **架构持久性**: 二十年来核心架构保持不变，证明了原始设计的可扩展性和前瞻性。所有新功能都可以追溯到"简单架构 + 灵活存储 + 性能关注"这三个基本原则
2. **渐进式创新**: 通过异步处理（非前台写路径）添加新功能（复制、CDC、物化视图），不影响关键路径性能。BEAR 机制巧妙解决了零停机添加副本的难题
3. **生产级可靠性**: 端到端校验 + 文件安装前完整验证 + 持续检查，使得"多年来几乎未观察到 SSTable 不一致"。大规模生产环境（10EB）下的可靠性实践极具参考价值
4. **资源效率意识**: 从 autosizing/autoscaling 到 cache right-sizing 到分层存储，系统性地考虑了"用更少做更多"的三个维度（减少预留、减少使用、资源分级）

### 局限性

1. **单行事务限制**: ACID 仅保证单行，跨行操作需要应用层自行处理。虽然论文指出许多应用可以"舒适地"使用单行事务，但这确实限制了适用场景
2. **最终一致性**: 复制仅提供最终一致性，跨行/跨 replica 无一致性保证。对于需要强一致性的场景，需要使用 [[Spanner]] 等其他系统
3. **经验报告性质**: 论文是经验总结而非严格实验评估，缺少与其他系统的定量对比数据（如与 DynamoDB、Cassandra 的性能对比），缺少具体的延迟/吞吐数字
4. **技术细节有限**: 论文明确指出"由于篇幅限制，只能突出部分工作，且只进入中等技术细节"，许多关键机制（如 rebalancer 算法、cache sizer 的 TCO 模型）未深入展开

### 潜在改进方向

1. 跨行事务支持或与 Spanner 更深度的集成，覆盖更多一致性需求场景
2. 探索更强的复制一致性模型（如 causal consistency），在保持低成本的同时提供更强保证
3. 更细粒度的资源调度，如基于查询复杂度的自动扩缩容，而非仅 CPU/RAM 驱动
4. 原生向量搜索支持，适应 AI 时代的工作负载趋势

### 可复现性评估

- [ ] 代码开源（Google 内部系统，不开源）
- [ ] 预训练模型（不适用）
- [x] 训练细节完整（系统架构和配置描述清晰）
- [ ] 数据集可获取（Google 内部生产数据）

---

## 关联笔记

### 基于
- [[Original Bigtable Paper|Bigtable: A Distributed Storage System for Structured Data]] (OSDI 2006): 本文的前置工作，奠定了核心架构
- [[LSM-tree]]: Bigtable 的底层存储结构
- [[SSTable]]: Bigtable 的持久化数据格式
- [[WAL]]: Bigtable 的持久化保证机制

### 对比
- [[DynamoDB]]: Amazon 的非关系型云数据库服务，同样是长期运行经验总结论文
- [[HBase]]: 开源的 Bigtable 克隆，Java 实现
- [[Cassandra]]: Facebook 开发的去中心化非关系数据库，数据模型类似但有差异（nested column family、CQL）
- [[Spanner]]: Google 的全球分布式数据库，提供强一致性和 ACID 事务

### 方法相关
- [[Chubby]]: 分布式锁服务，存储 Bigtable 启动元数据
- [[Colossus]]: Google 分布式文件系统，Bigtable 的底层存储
- [[CRDT]]: 冲突无关复制数据类型，用于计数器和聚合
- [[Bloom Filter]]: 空间高效的概率数据结构，优化读路径
- [[Coprocessor]]: Bigtable 的内部执行层，支持 server-side 逻辑
- [[GoogleSQL]]: Google 内部使用的 SQL 方言
- [[MapReduce]]: 大规模并行计算框架
- [[Eventual Consistency]]: Bigtable 的复制一致性模型
- [[HTAP]]: 混合事务/分析处理
- [[Change Data Capture]]: 实时数据变更捕获
- [[Materialized Views]]: 预计算视图

### 硬件/数据相关
- [[Tablet]]: Bigtable 的数据托管/移动/均衡单位
- [[Locality Group]]: Bigtable 列存储分组，压缩单位
- [[Tablet Server]]: Bigtable 的数据服务节点

---

## 速查卡片

> [!summary] Twenty Years of Bigtable
> - **核心**: Google Bigtable 二十年演进回顾，10EB 数据、70亿 QPS
> - **架构**: LSM-tree + WAL + Tablet/Master + Colossus 分布式文件系统
> - **新功能**: 多主复制(最终一致性) / SQL查询 / CDC / CRDT计数器 / 物化视图
> - **优化**: 外部压缩 / 行缓存 / 分片Bloom Filter / 自动扩缩容 / 端到端校验
> - **启示**: 简单可扩展架构 + 异步非前台处理 = 持久生命力

---

*笔记创建时间: 2026-07-09 00:30*
