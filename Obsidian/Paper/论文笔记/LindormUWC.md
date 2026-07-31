---
title: "Lindorm-UWC: An Ultra-Wide-Column Database for Internet of Vehicles"
method_name: "LindormUWC"
authors: [Qianyu Ouyang, Chunhui Shen, Wenlong Yang, Peng Yu, Qiang Xiao, Jianhui Lei, Yadong Chen, Qilu Zhong, Xiang Wang, Yong Lin, Qingyi Meng, Zhicheng Ji, Wei Meng, Cen Zheng, Sheng Wang, Dan Pei, Wei Zhang, Feifei Li, Jingren Zhou]
year: 2024
venue: PVLDB 17(12)
tags: [wide-column-database, internet-of-vehicles, cloud-native, lsm-tree, hybrid-storage, lindorm]
zotero_collection: VLDB/2024
image_source: local
created: 2026-07-31
---

# 论文笔记：Lindorm-UWC: An Ultra-Wide-Column Database for Internet of Vehicles

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | Alibaba Cloud、清华大学、浙江大学 |
| 日期 | 2024，PVLDB 17(12): 4117–4129 |
| 项目主页 | — |
| 对比基线 | [[MongoDB]]、[[HBase]]、[[InfluxDB]] |
| 链接 | [DOI](https://doi.org/10.14778/3685800.3685831) |
| 原始 PDF | `VLDB/2024/Lindorm-UWC.pdf` |

---

## 一句话总结

> Lindorm-UWC 面向车联网单车数千指标、分批更新与混合列选择查询，结合 Merge-on-Read、行式 SSTable + Parquet 双格式、时间窗口冷热分层及 Region 热点治理，在高写吞吐与全行/少列查询之间取得平衡。

---

## 核心贡献

1. **IoV 超宽列负载刻画**：以三家真实车联网公司的写入与查询日志证明：一行可有数千指标、同一行分多次写入、全列和少列查询并存。
2. **UWCSE 存储引擎**：设计 `merged KV item` 与 [[Merge-on-Read]]，将同一 `(VehicleID, timestamp)` 的分批指标写入转化为适合 [[LSM-tree]] 的更新路径。
3. **行列双格式**：为每个行式 [[SSTable]] 同步构建一个 [[Parquet]] 文件；全行/多指标走行式，少指标走列式，控制读放大。
4. **云原生弹性与分层**：以 LDServer/LDMaster/共享文件系统实现计算存储分离，并通过 region swap、cascaded split、时间窗口 compaction 和冷数据索引缓存处理热点与成本。

---

## 问题背景

### 要解决的问题

[[Internet of Vehicles|车联网]] 会以高频率采集车辆遥测。其主键通常是 `(VehicleID, timestamp)`，但单行包含的速度、位置、轮胎、雷达、电池等 metrics 可超过 2500 个。论文面向的生产流量可超过 **1 GB/s**、每天约 **80 TB**，并在 2019 年起于阿里云服务 PB 级 IoV 数据。

### 真实负载发现

1. **超宽行**：公司 A、B、C 的总 metric 数分别为 2999、2083、260（Table 2）。
2. **分批写入**：公司 A 每次只写平均 223 个指标，仅约全行的 7.4%，同一逻辑行约分 14 次到达；C 也存在约 6 次更新式写入。
3. **查询两极**：A 常取 ≤130 个指标，B/C 中大量查询读取全部指标；因此单一行存或列存都无法普遍最优。
4. **近期短窗口**：超过 94% 的 B/C 查询关注最近 1 小时；A 的 93.6% 查询关注一周内。大部分查询是单车、短时间范围查询。

### 现有方法的局限

- 面对数千列，按 metric 分开写会造成巨大的键、索引与元数据放大。
- 对同一逻辑行的 partial update，传统“读出整行再重写”会放大写 I/O。
- 行存擅长全行读，少列查询会读入无关数据；列存擅长少列读，但高频、分批更新时维护昂贵。
- 热点车辆或主键范围会使分区负载失衡；旧数据又需要从高性能 SSD 下沉到低成本 HDD/OSS。

### 本文的动机

IoV 工作负载呈现“高写、低读、很少删除/修改”的特点，且 query 的列选择性强烈分化。论文因此不追求单格式统一，而选择 **写路径友好的行式 LSM + 查询路径可选的列式副本**，并把合并、冷热、热点治理做成引擎内部能力。

---

## 方法详解

### 系统架构

Figure 5 中 Lindorm-UWC 由三部分组成：

- **LDServer**：主要计算节点，管理数据并处理读写请求。
- **LDMaster**：管理负载均衡和故障恢复，主备部署，依赖 ZooKeeper 高可用。
- **共享分布式文件系统**：所有 LDServer 可访问的持久化存储。

论文明确指出 UWC 是一个 Lindorm engine，并与其他引擎共享底层文件系统。官网中 [[LindormDFS]] 是 Lindorm 的共享文件系统产品名；将二者对应是合理的产品架构映射，但本文 Figure 5 只写作 “shared distributed file system”。同理，UWC 的动态列、Region、LSM 与官网 [[LindormTable]] 高度一致，但论文没有直接声称“UWC 等同于全部 LindormTable 实现”。

### 模块 1：数据模型与 Region

一行由主键 `(VehicleID, timestamp)` 和动态 metrics 组成。请求先根据主键范围路由到 region，再由 region 所属 LDServer 处理。

| 组成 | 语义 |
|---|---|
| Row key | `(VehicleID, timestamp)` |
| Metric columns | 可动态扩展的指标列，例如 speed、battery、location |
| Region | 连续主键范围对应的逻辑分区 |
| LDServer | 执行该 region 的读写、内存管理与文件生成 |

### 模块 2：UWCSE 写入、Merged KV 与 Merge-on-Read

#### 写入路径

1. 请求路由至目标 region / LDServer。
2. 先追加 [[WAL]]，确保故障恢复所需的持久性。
3. 在 Memtable 中，把同一请求的多 metric 合并为一个 **merged KV item**：
   - key：`(VehicleID, timestamp)`；
   - value：含 type/name/value 的扁平化多指标序列。
4. Memtable flush 成行式 SSTable，并同时生成对应 [[Parquet]] 列式文件。
5. 后续 compaction 重建新的 SSTable + Parquet 文件。

#### Merge-on-Read

同一 row key 若因分批上报出现多个 value，不在每次写入时重写完整行；而是在读取或 compaction 时将各部分指标拼接合并。可写成下列语义：

$$
V(k)=v_1 \sqcup v_2 \sqcup \cdots \sqcup v_m,\quad k=(\operatorname{VehicleID},t)
$$

**含义**：同一主键 $k$ 的多个部分指标值 $v_i$ 在读取/合并阶段做字段并集 $\sqcup$；若有相同 metric 的冲突，则遵从系统版本顺序。该式是对论文 Merge-on-Read 语义的说明性表示，不是原文编号公式。

**符号说明**：
- $k$：车辆 ID 与时间戳构成的行键。
- $v_i$：一次分批上传得到的指标子集。
- $V(k)$：逻辑完整行。

该机制尤其适合高写低读、无删除/少修改的 IoV 数据；它并不意味着适用于任意高频随机更新 OLTP 场景。

### 模块 3：SSTable 与行列双格式

行式 [[SSTable]] 由固定大小 Data Block、Index Block、Bloom Block 与 Root Block 构成：

| 结构 | 内容 | 作用 |
|---|---|---|
| Data Block | 排序 KV 与压缩数据 | 主数据读取 |
| Index Block | 每个 Data Block 的起始 key/offset | 快速定位数据块 |
| Bloom Block | 分区 [[Bloom Filter]] | 排除不可能包含 key 的文件/块 |
| Root Block | Index/Bloom 的位置与文件元数据 | 保存时间范围、metric set 等元信息 |

每个 SSTable 有一个 1:1 的 [[Parquet]] 文件：

- **全行 / 多指标读取**：优先读行式 SSTable，避免跨许多列文件拼装行。
- **少指标读取**：可指定走 Parquet，仅扫描目标列，减少无关 I/O。
- **代价**：写入与 compaction 要维护双格式，属于以写资源换查询选择性的设计。

### 模块 4：Region swap 与 Cascaded split

#### Region swap

系统以 region 的 CPU 消耗衡量负载，选取高/低负载 LDServer 进行 region 交换并迭代，直到负载差低于阈值。其优化目标可概念化为：

$$
\min_{\pi}\ \max_j L_j(\pi)-\min_j L_j(\pi)
$$

**含义**：通过 region 到 LDServer 的分配 $\pi$，缩小各节点负载 $L_j$ 的最大差异。该式是对 Figure 6 算法目的的形式化描述，不是论文原始公式。

#### Cascaded split

对热点 region，系统从请求采样中选择 split point。若 split 过程中的子 region 又成为热点，允许其继续 split，并以“引用的引用”关系追踪多层分裂，减少同步等待和迁移阻塞。

### 模块 5：时间窗口 compaction 与冷热分层

- 按时间窗口执行 compaction，只合并同一个窗口内的文件；降低跨时间窗口的写放大。
- 当时间窗口冷却后，合成为更少的 SSTable 并下沉到 HDD/OSS 等冷存。
- 热存中保留冷文件的 metadata、[[Bloom Filter]] 和 index cache；查询先在热层过滤，只有必要时才访问冷层。

这与 [[Hot-Cold Tiering]] 和官网 LDFS 的多规格混合存储思想相符。

### 可靠性与恢复

- WAL 是持久化先行路径；LDMaster 负责故障恢复与负载管理。
- LDMaster 主备部署并以 ZooKeeper 协调高可用。
- 论文未详细说明多副本复制协议、线性一致性与事务隔离级别，不能从其架构图推断出通用 ACID 语义。

---

## 关键公式

论文没有编号数学公式。上文的 Merge-on-Read 合并式与 Region 负载目标均为依据论文机制给出的说明性形式化表达，用于明确语义，不应视为原文推导。

---

## 关键图表

> UWC PDF 图形主要为矢量对象；所有含图页面均已以高分辨率 PNG 渲染至 `assets/LindormUWC/`，完整保留论文中 11 个 Figure 和 8 个 Table 的原始图表、坐标与 caption。

### Figure 1：月度写入吞吐（论文 p4）

![[assets/LindormUWC/LindormUWC_page4.png]]

**说明**：三家 IoV 系统的写操作数与数据吞吐分布。A 平均约 721 MB/s，B 平均约 907 MB/s、峰值 1.68 GB/s，说明系统必须承受持续高带宽摄入。

### Figure 2：目标指标数的累积分布（论文 p4）

![[assets/LindormUWC/LindormUWC_page4.png]]

**说明**：不同公司的每次查询所取指标数差异极大：A 偏少列，B/C 常见全行读取。

### Figure 3：查询数据新鲜度分布（论文 p5）

![[assets/LindormUWC/LindormUWC_page5.png]]

**说明**：$t-t_s$ 说明查询起始时间距离当前的远近；业务显著偏向近期数据。

### Figure 4：查询时间范围分布（论文 p5）

![[assets/LindormUWC/LindormUWC_page5.png]]

**说明**：$t_e-t_s$ 描述查询窗口长度，绝大部分为短时间范围，强化了近期热点与冷数据分层的必要性。

### Figure 5：Lindorm-UWC 总体架构（论文 p6）

![[assets/LindormUWC/LindormUWC_page6.png]]

**说明**：LDServer 处理请求，LDMaster 进行均衡与恢复，数据位于共享分布式文件系统。

### Figure 6：Region swap（论文 p7）

![[assets/LindormUWC/LindormUWC_page7.png]]

**说明**：高低负载 LDServer 交换 region 以缩小 CPU 负载差，避免只靠单向迁移带来的临时失衡。

### Figure 7：热点 Region 的 cascaded split（论文 p8）

![[assets/LindormUWC/LindormUWC_page8.png]]

**说明**：region O 分裂为 A/B 后，若 B 再成为热点，可继续产生后代 region；引用链使迁移与请求处理不被单次分裂强耦合。

### Figure 8：UWCSE 总览（论文 p8）

![[assets/LindormUWC/LindormUWC_page8.png]]

**说明**：写请求经 WAL、Memtable、行式 SSTable 与列式 Parquet 文件；查询可按访问列数选择数据格式。

### Figure 9：SSTable 结构（论文 p8）

![[assets/LindormUWC/LindormUWC_page8.png]]

**说明**：Root/Index/Bloom/Data Blocks 组织文件元数据、过滤信息和有序 KV 数据。

### Figure 10：时间窗口冷热管理（论文 p9）

![[assets/LindormUWC/LindormUWC_page9.png]]

**说明**：系统周期性生成热时间窗口；旧窗口在合并后下沉，热层保留冷文件的索引和元数据。

### Figure 11：不同 split 与 metric scale 下的写吞吐（论文 p10）

![[assets/LindormUWC/LindormUWC_page10.png]]

**说明**：update-style 写入（更大的 split）下，Lindorm-UWC 仍保持高吞吐，MongoDB 等基线下降明显。

### Table 1：IoV 数据样例（论文 p3）

**说明**：展示单车在不同 timestamp 的多指标记录；原始列和样例值完整保留在 p3 渲染页。

### Table 2：三家公司的 Metric 规模（论文 p3）

| 公司 | 总 metrics | 单次写入 metrics | 近似一行分批次数 |
|---|---:|---:|---:|
| A | 2999 | 223 | 14 |
| B | 2083 | 2083 | 1 |
| C | 260 | — | 约 6 |

### Table 3：查询概览（论文 p4）

**说明**：A/B 约 100%、C 为 99.79% 的查询是单车查询；具体时间新鲜度与窗口统计完整保留在 p4 渲染页。

### Table 4：不同数据库写吞吐（ops/s，论文 p5）

**说明**：动机实验表明当 metrics 增至 2000/3000 时，MongoDB、HBase、InfluxDB 写入吞吐均明显下降。例如 2000 metrics 时分别为 7752、9651、4230 ops/s；完整表见 p5 渲染页。

### Table 5：不同数据库查询吞吐（QPS，论文 p6）

**说明**：动机实验显示：MongoDB 在全行读取上较好但少列收益有限；InfluxDB 少列高但多指标显著下降；HBase 整体偏低。完整原表见 p6 渲染页。

### Table 6：Lindorm-UWC 数据模型（论文 p6）

**说明**：包含 row key、timestamp 与动态 metric 列的模型示例；完整表见 p6 渲染页。

### Table 7：查询吞吐（QPS，论文 p11）

| 时间范围 | 指标数 | Lindorm-UWC | MongoDB | HBase | InfluxDB |
|---|---:|---:|---:|---:|---:|
| 10 min | 2000 | 173 | 108 | 9.84 | 11.5 |
| 10 min | 100 | 217（列式 74） | 222 | 21.5 | 109 |
| 10 min | 20 | 421 | 243 | 25.7 | 554 |
| 10 min | 1 | 7524 | 244 | 39.6 | 7093 |
| 60 min | 2000 | 27.0 | 16.9 | 1.73 | 32.5 |
| 60 min | 100 | 31.3（列式 51.2） | 33.3 | 3.37 | 50.5 |
| 60 min | 20 | 219 | 35.9 | 64.3 | 240 |
| 60 min | 1 | 1474 | 36.3 | 97.0 | 1716 |

**说明**：括号值为用户指定列式路径。2000 指标时 UWC 约为 MongoDB 的 1.6×；1 指标时约为 MongoDB 的 30.8×。列式并非在所有列数下最优，格式选择仍取决于列选择性与窗口。

### Table 8：2000 Metrics 写入时的系统负载（论文 p11）

| 配置 | CPU | Disk throughput | Disk IOPS |
|---|---:|---:|---:|
| 仅行式 | 67% | 322 MB/s | 3066 |
| 行式 + Parquet | 88% | 373 MB/s | 3365 |

**关键发现**：维护列式副本使 CPU 相对增加约 31.3%、磁盘吞吐增加 15.8%、IOPS 增加 9.8%，这是为少列查询能力付出的可量化成本。

---

## 实验

### 数据集与工作负载

| 项目 | 内容 |
|---|---|
| 真实日志 | 三个 IoV 公司：一个月写入日志、一周查询日志 |
| 合成数据 | 10,000 辆车，可调 `metric_scale` 与 `split` 模拟超宽行和分批写入 |
| 读负载 | 单车、10/60 分钟时间窗口、1/20/100/2000 指标查询 |

### 实现细节

- **部署**：阿里云产品形态。
- **数据库集群**：每个系统 3 节点，每节点 8 cores / 64 GB RAM。
- **客户端**：128 cores / 512 GB RAM。
- **基线**：[[MongoDB]]、[[HBase]]、[[InfluxDB]]。
- **变量**：`metric_scale` 控制行宽，`split` 控制一行被拆成多少批写入。

### 关键实验结论

1. 无 update-style（`split=1`）时，Lindorm-UWC 平均写吞吐比 MongoDB 高 **79%**，对 HBase/InfluxDB 达到一个数量级以上优势。
2. 有分批更新时，MongoDB 吞吐显著衰减；Merge-on-Read 让 UWC 能保持高写入性能。
3. UWC 的行式路径适合全行/多指标读取；少量指标可用 Parquet 降低读放大，达到远超 MongoDB 的 QPS。
4. 双格式并非“免费”：Table 8 明确给出 CPU、带宽与 IOPS 的额外开销。
5. region swap 与 cascaded split 是处理 IoV 热点主键的关键弹性机制；论文将其与共享存储架构共同用于降低迁移影响。

---

## 批判性思考

### 优点

1. **问题定义扎实**：先用真实 IoV 写入和查询日志量化“超宽 + 分批写 + 两极查询”的矛盾。
2. **没有强行单格式统一**：行式与列式分别服务不同 query shape，技术取舍透明。
3. **更新语义贴合负载**：Merge-on-Read 把“分批上报”转成更适合 LSM 的写入模式。
4. **涵盖完整系统问题**：除存储布局外还包括热点 Region、冷存、恢复和共享存储。

### 局限性

1. **格式选择需手工指定**：论文明确指出目前由用户选择行式/列式路径，尚无自动代价决策。
2. **Merge-on-Read 假设有限**：最适合高写、低读、无删除/少修改；高读频率或频繁字段覆盖可能增加读时合并成本。
3. **评估范围偏 IoV**：主键、时间窗口与单车查询高度特化，复杂 join、跨车聚合和通用 SQL 未充分评估。
4. **写放大未完全展开**：双格式的开销只在一个固定负载点报告；更大行宽、模式频繁变化或更高 compaction 压力下的成本未知。
5. **一致性协议细节缺失**：未完整描述复制、事务隔离或跨机房一致性。

### 潜在改进方向

1. 以查询统计和文件列集合元数据构建 [[Row-Column Cost Model|行列代价模型]]，自动选择或自适应维护格式。
2. 对 Merge-on-Read 引入读放大预算、后台 materialization 与热点行预合并。
3. 评估与 [[LindormSearch]] 的透明索引协同，以及官网 [[LindormStream]] / LTS 的实时数据通道。
4. 研究面向 schema 演化和字段覆盖写的增量列式维护策略。

### 可复现性评估

- [ ] 代码开源
- [ ] 完整产品部署与配置
- [x] 工作负载变量、硬件和对比系统说明
- [x] 基于真实 IoV 日志的特征分析

---

## 关联笔记

### 基于

- [[LSM-tree]]：Memtable、SSTable 与 compaction 的基础结构。
- [[SSTable]]：UWCSE 行式文件的主体。
- [[Parquet]]：面向少列查询的列式副本。
- [[WAL]]：写入持久性与恢复基础。
- [[Shared-Storage Architecture]]：LDServer 与共享文件系统解耦的架构基础。

### 对比

- [[MongoDB]]：文中主要文档数据库基线。
- [[HBase]]：宽列数据库基线。
- [[InfluxDB]]：时序数据库基线。
- [[LindormTSDB]]：同一 Lindorm 体系的监控时序引擎，优化目标不同。

### 方法相关

- [[Internet of Vehicles]]：本文的目标业务域。
- [[Ultra-Wide Column]]：数千动态 metrics 的数据模型。
- [[Merge-on-Read]]：处理 update-style writing 的核心机制。
- [[Hybrid Row-Column Storage]]：SSTable 与 Parquet 的双格式策略。
- [[Region Swap]]：基于负载的 Region 重平衡。
- [[Cascaded Split]]：热点 Region 的递归分裂机制。
- [[Time-Window Compaction]]：按时间窗口限制 compaction 边界。
- [[Hot-Cold Tiering]]：近期热数据和历史冷数据的成本分层。

### 产品架构映射（需要区分论文事实与产品推断）

- [[LindormDFS]]：论文的 shared distributed file system 与官网 LDFS 相对应；论文直接事实是“共享文件系统”，产品名对应是架构映射。
- [[LindormTable]]：UWC 的动态列、Region、LSM 与宽表引擎高度一致，但论文未把二者直接定义为同一实现。
- [[LindormStream]]：官网流引擎能力；UWC 论文未描述直接集成路径。
- [[LindormSearch]]：官网可为宽表提供透明索引；UWC 论文未评测这一协同。

---

## 速查卡片

> [!summary] Lindorm-UWC
> - **核心**: 面向 IoV 超宽行与分批指标写入，兼顾全行和少列查询。
> - **方法**: merged KV、Merge-on-Read、SSTable + Parquet 双格式、region swap/cascaded split、时间窗口冷热分层。
> - **结果**: `split=1` 写吞吐平均高于 MongoDB 79%；1 指标查询约为 MongoDB 30.8×；双格式代价可量化。
> - **代码**: 未开源。

---

*笔记创建时间: 2026-07-31*
