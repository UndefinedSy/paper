---
title: "GaussDB-Vector: A Large-Scale Persistent Real-Time Vector Database for LLM Applications"
method_name: "GaussDBVector"
authors: [Ji Sun, Guoliang Li, James Pan, Jiang Wang, Yongqing Xie, Ruicheng Liu, Wen Nie]
year: 2025
venue: PVLDB
tags: [vector-database, ann-search, hybrid-search, distributed-database, rag, llm]
zotero_collection: VLDB/2025
image_source: local
created: 2026-08-07
---

# 论文笔记：GaussDB-Vector

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | Tsinghua University；Huawei |
| 发表 | PVLDB 18(12), 2025，pp. 4951–4963 |
| DOI | [10.14778/3750601.3750619](https://doi.org/10.14778/3750601.3750619) |
| 对比基线 | [[Milvus]]、[[Elasticsearch]]、[[PGVector]] |
| 适用场景 | [[RAG]]、语义缓存、LLM Coding Copilot、持续更新的向量检索 |

---

## 一句话总结

> GaussDB-Vector 将可更新的磁盘 [[Vamana]] 图索引、I/O 感知页布局、标量—向量混合索引和分布式路由整合进关系型数据库，实现实时持久化 ANN 检索。

---

## 核心贡献

1. **关系型实时向量数据库架构**：在 CN/DN 两层分布式 GaussDB 框架中支持 SQL、事务、复制、立即可见的插入以及标量—向量联合检索。
2. **I/O 优化的 ANN 索引与更新**：同时实现 [[Inverted File Index|IVF]] 与 [[Vamana]]；为图索引按维度/数据量选择三种页面布局，缓存热点节点，并将插入邻居选择最坏复杂度从 DiskANN 的 $O(m^3)$ 降至 $O(m^2)$。
3. **自适应混合检索**：以平衡树组织标量键，在根/中间节点附带向量子索引；低选择性走叶页小扫描，高选择性走最小超集子索引，兼顾稳定查询延迟与更新成本。
4. **分布式与硬件协同**：按向量距离分片、基于基数估计选择 DN、检测分布漂移并重分布；支持 SIMD、GPU 与 Ascend NPU 加速。

---

## 问题背景

### 要解决的问题

LLM 的 [[RAG]] 需要把知识 embedding 当作长期记忆，以降低幻觉；语义缓存则希望复用 embedding，减少推理调用。数据库因此必须同时支持十亿级持久化向量、低延迟高召回、频繁增删改、[[ACID]]、高并发、跨机扩展及标量过滤。

### 现有方法的局限

- [[Milvus]] / [[Elasticsearch]] 以段式组织数据：查询要搜索并合并多个段；段合并/拆分的索引重建昂贵；新写入先缓存在内存，数据新鲜度非零延迟。
- [[PGVector]] 将索引落在 PostgreSQL 页上，具备数据库能力，但其磁盘 [[HNSW]] 随机 I/O 多，且原生单机，难以扩至十亿—百亿级。
- 常规 ANN 索引通过扩大候选集做标量过滤，在低选择性条件下计算与访问冗余严重。

### 本文的关键主张

论文并非提出一个新的距离度量或 ANN 图，而是把已有 Vamana/DiskANN 思路改造成**可维护的数据库物理结构**：图节点布局要匹配页和维度，增删改要与事务/可见性协作，过滤要变为联合索引，分布式搜索要减少 fan-out。

---

## 方法详解

### 两层系统架构

- **Coordinate Node (CN)**：接收 SQL/SDK 请求，进行解析、优化、数据路由和查询路由。
- **Data Node (DN)**：执行算子与数据修改，管理页式存储、索引和硬件加速。
- **Cluster Manager / Global Transaction Manager**：分别负责集群编排和分布式事务 ID。
- **SQL 接口**：提供 `floatvector`、`boolvector`、`sparsevector`，支持 `CREATE INDEX USING IVF|VAMANA`、`ORDER BY distance LIMIT k`、并发建索引和标准 DML。

### IVF 索引

[[Inverted File Index|IVF]] 将最近向量聚集至物理连续块：元页记录配置；聚类页存中心；数据页将一个 cluster 的向量与元数据放在一起；可选 [[Product Quantization|PQ]] 表及 PQ code 页压缩距离计算/内存占用。删除采用逻辑标记，达到阈值后 vacuum；每个 cluster 的页链在重写时复制存活向量并回收旧页，降低碎片。

### 持久化 Vamana 图索引

索引文件由 metapage、PQTable 与图页构成。论文针对维度和规模选择三种布局：

1. **低维（示例阈值 130）**：当前节点的原始向量、PQ code、邻居及其 PQ code 共置一页，一次 I/O 取到邻居估值。
2. **中高维（示例阈值 1565）**：图数据页仅存邻居 tuple ID，PQ code 独立成 zone；PQ 总体较小，易于缓存。
3. **超大规模/极高维**：节点与边完全分离且紧凑排列；启用 PQ 时可从索引中移除原始向量。

构建阶段将样本聚为 $N_g$ 个簇，每个向量归属其最近的两个簇；各子图在内存中以 48 出度建图，合并入磁盘全图时为 96 出度。双簇归属保留跨子图连通性，而分块建图避免全量建图的频繁换页。

**实时插入与删除**：插入先图搜索候选邻居，再用逐步放宽的 $\alpha'$（上限 1.2）筛选；回边更新中使用“最远邻居”启发式线性扫描，避免 DiskANN 对每个邻居做二次全对比较，因而从 $O(m^3)$ 降为 $O(m^2)$。删除通过 `ctid → tid` 倒排表精确定位图节点并逻辑标记；更新等价于删除加插入。

**Vacuum**：死/更新行超过 10% 时，并行识别死节点；阻止新插入并等待旧插入结束；重插受死节点影响的邻居以修复边；最后等待读/删结束后物理回收，避免读到复用页并修复不可达节点。

### 标量—向量混合索引

[[Hybrid Search|混合索引]]是多列物理索引：标量键为页式平衡树，根与中间节点连接向量子索引，叶节点只紧凑保存 `<scalar, vector, ctid>`。查询选择策略为：

- 标量条件命中 **>50%** 数据时，直接查覆盖该条件最小超集的内部节点向量子索引；
- 低选择性时，定位叶节点，对通常少于 100 行的向量顺序计算/排序；
- 图子索引只放在根或拥有超过一千万向量的中间节点；叶节点无子索引以避免频繁 split 的维护代价。

插入可触发 split；相邻节点有空位时使用 rotate。该设计明确是在“更多子索引带来更快搜索”与“子索引重建使更新更贵”之间折中。

### 分布式检索

向量按距离聚类，每个 cluster 对应一个 DN，centroid 存在 CN：初始 RANDOM；达到如 50,000 条后进入 FULL REDISTRIBUTING；稳定后 CLUSTERED，新增向量写入最近 DN、旧数据后台迁移。后台采样重新标注；若漂移向量比例超过如 10%，更新中心并重分布。

路由根据 `ANALYZE` 训练的 DN 基数模型决定 fan-out：范围查询利用距离阈值；top-$k$ 用二分搜索距离阈值，使各 DN 估计基数之和超过 $k$；混合查询还考虑标量选择率。对边界向量额外选择距离接近的 cluster，减轻估计误差导致的召回损失。

### 硬件加速

CPU 侧用 SIMD 和线程并行。NPU 侧将向量 batch 成矩阵，使用

$$
D(x, y)=x^2-2(x\cdot y^T)+y^2
$$

把距离计算转为矩阵运算；NPU 计算 centroid 更新与批量距离，CPU 并行构图。论文指出 IVF 更易获益，图搜索中跨设备数据搬移可能抵消加速收益。

---

## Algorithm 1–5

| Algorithm | 作用 | 关键点 |
|---|---|---|
| BuildGraphIndex | 构建全局图 | 计算 PQ、双最近簇归属、以子图建图后合并 |
| BuildSubgraph | 并行构建内存子图 | 逐个插入随机初始化图 |
| VectorInsert | 增量插入 | 动态放宽 $\alpha'$，最远邻居启发式降低复杂度 |
| VectorSearch | 贪心 ANN 搜索 | 有限候选 min-heap，直到无未访问节点 |
| Index_Vacuum | 清理死节点 | 两阶段锁与邻居重插，随后安全物理回收 |

---

## 关键图表

> 注：以下 15 个嵌入项分别引用原论文相应页面的无裁剪渲染，以完整保留图中算法、坐标轴与图例；同页多图在视觉上仍逐一对应原 Figure 编号。

### Figure 1：系统架构
![[GaussDBVector_fig1.png]]
CN 负责优化与路由，DN 包括类型、向量接口、页管理、聚类/PQ/相似度/图、并行与本地/全局缓存模块。

### Figure 2：IVF 索引结构
![[GaussDBVector_fig2.png]]
展示 metapage、聚类页、数据页和可选 PQ table/code page 的布局。

### Figure 3：图索引三种存储结构
![[GaussDBVector_fig3.png]]
低维共置、PQ 独立区、节点—边分离三种图页设计对应不同维度与容量。

### Figure 4：标量—向量混合索引
![[GaussDBVector_fig4.png]]
平衡树的内部节点含 IVF/Graph 子索引；叶页保存 `<id, vec, ctid>`，并展示 split 与 rotate。

### Figure 5：分片与查询调度
![[GaussDBVector_fig5.png]]
新增向量路由到最近 DN，查询发往近邻 DN；漂移后从 DN2 迁移数据修复分片。

### Figure 6：单连接 ANN 延迟
![[GaussDBVector_fig6.png]]
SIFT、GIST、HUAWEINet 三数据集上，GaussDB-Vector 在相同 recall 下总体最低或并列最低延迟。

### Figure 7：多连接 ANN 延迟
![[GaussDBVector_fig7.png]]
50 并发下 GaussDB-Vector 延迟优势仍保持，特别是 HUAWEINet 上与基线拉开差距。

### Figure 8：多连接 ANN QPS
![[GaussDBVector_fig8.png]]
展示相同 recall 维度下的吞吐比较；论文归因于更少计算和随机 I/O。

### Figure 9：插入尾延迟
![[GaussDBVector_fig9.png]]
实时系统中 GaussDB-Vector 的 p99/max 优于 PGVector；Milvus/Elasticsearch 的段式批量构建造成高尾延迟。

### Figure 10：不同标量选择率下的混合查询
![[GaussDBVector_fig10.png]]
90%、50%、10% selectivity 下混合索引均稳定；PGVector 在低选择率时扫描冗余候选而显著退化。

### Figure 11：总体混合查询延迟
![[GaussDBVector_fig11.png]]
随机标量—向量查询上 GaussDB-Vector 最优，支持“按节点选择子索引”而非单一过滤策略。

### Figure 12：自动递增 ID 下的插入
![[GaussDBVector_fig12.png]]
进一步显示不同数据集上的 p50/p95/p99/max 插入表现。

### Figure 13：横向扩展
![[GaussDBVector_fig13.png]]
SIFT-10B 上机器数增加时延迟下降、吞吐提高；40 台约 60ms、1500 QPS、99% recall。

### Figure 14：持续更新吞吐
![[GaussDBVector_fig14.png]]
更新吞吐在 vacuum 期间短暂下降，最大回归约 5%–15%，之后恢复。

### Figure 15：Ascend NPU 加速
![[GaussDBVector_fig15.png]]
IVF 构建获益最明显；图构建的 CPU↔NPU 传输和难并行性限制总体加速。

### Table 1：数据集

| Dataset | #rows | #dim | #cols | scalar ndv | distance |
|---|---:|---:|---:|---:|---|
| SIFT | 10M | 128 | 1 | — | Euclidean |
| GIST | 1M | 960 | 1 | — | Euclidean |
| HUAWEINet | 15M | 1024 | 1 | — | Cosine |
| SIFT-H | 10M | 128 | 2 | 1M | Euclidean |
| GIST-H | 1M | 960 | 1 | 500K | Euclidean |
| HUAWEINet-H | 10M | 1024 | 1 | 1M | Cosine |
| SIFT-10B | 10B | 128 | 1 | — | Euclidean |

---

## 实验与证据

- **环境**：40 台机器；每台 72 核 Intel 3.0GHz、64GB 内存、2TB 磁盘、10GbE；其中一台有 8 张 Ascend-920B NPU。
- **索引体积**：10M SIFT 为 9GB；10B SIFT 为 8.6TB；10M HUAWEINet 为 35GB；作者称与 Milvus/Elasticsearch 可比。
- **ANN**：单机十亿级承诺 <50ms、>95% recall；多机目标为十亿—百亿规模、>100,000 QPS。论文图上在所测 SIFT/GIST/HUAWEINet 中相较基线呈 1–5× 优势，但该倍数是论文总体声明而不是每一条曲线均达到。
- **扩展性**：SIFT-10B 测试在 40 台机器约 60ms、1500 QPS、99% recall；少于 30 台时性能受每机数据量和查询调度影响。
- **真实场景**：服务华为 XiaoYi Search；Coding Copilot 将代码片段、文档及 embedding 放入系统，峰值检索约 5,000 QPS，更新约 3,000 QPS，平均检索+更新在 100ms 内。

---

## 批判性思考

### 优点
1. **系统闭环完整**：将索引、页存储、更新/回收、事务、SQL、过滤、分片和加速一起讨论，适合评估生产化向量数据库，而非只比较 ANN 曲线。
2. **更新路径具体**：`ctid → tid` 定位、死节点重插、两阶段锁和 >10% vacuum 阈值明确覆盖了持久图索引常被忽略的维护问题。
3. **混合查询设计有针对性**：不同选择率走不同物理路径，避免“先 filter 后 ANN”或“先 ANN 后 filter”的单策略缺陷。

### 局限性
1. **可复现实证有限**：论文未给出可执行开源实现、参数全表或完整 cardinality 模型；许多系统工程效果难以独立复验。
2. **关键声明缺少细分归因**：1–5× 总体提升混合了布局、缓存、并行、剪枝、路由和硬件加速，缺少每个组件的消融量化。
3. **分布式基数估计风险**：边界向量和分布漂移被作者承认会放大估计误差；额外路由 cluster 改善 recall，但会牺牲 fan-out/尾延迟。
4. **NPU 增益并非普适**：图索引的跨设备数据传输可抵消计算加速，说明专用硬件不自动改善图 ANN。

### 可借鉴方向

- 若目标是事务型实时 RAG，优先评估“更新后立即进入索引”和 vacuum 期间 p99，而不仅是静态数据集 recall/QPS。
- 若标量过滤是高频工作负载，选择率驱动的多粒度子索引比盲目增加 ANN 候选集更有价值。
- 若部署多机，应该把路由的召回误差、边界查询和漂移重平衡当成索引质量的一部分测试。

---

## 关联笔记

- [[Vamana]]：本文图 ANN 的基础算法。
- [[Inverted File Index]] 与 [[Product Quantization]]：本文的另一类索引和压缩路径。
- [[Hybrid Search]]：本文以树 + 向量子索引实现的标量过滤方案。
- [[Vector Sharding]]：按距离聚类分片与基数驱动路由。
- [[MVCC]]：逻辑删除/可见性与安全页面回收的数据库语义基础。
- [[LindormVector]]：同为云原生分布式向量引擎，可用于对比“关系数据库内核整合”与“多模 NoSQL 内建向量服务”的设计取舍。

---

## 速查卡片

> [!summary] GaussDB-Vector
> - **核心**：面向 LLM/RAG 的实时持久化关系型向量数据库。
> - **索引**：IVF + 磁盘 Vamana；I/O 感知三布局和热点节点缓存。
> - **更新**：零延迟可见插入、逻辑删除、异步 vacuum 和图修复。
> - **过滤/分布式**：树形混合索引；距离聚类分片 + 基数估计路由。
> - **证据**：13 页论文含 15 图、1 表、5 算法；40 台实验中 10B SIFT 达约 60ms / 1500 QPS / 99% recall。

*笔记创建时间: 2026-08-07*
