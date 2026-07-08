---
title: "LindormVector: A Distributed Vector Engine on a Cloud-Native Multi-Model NoSQL Database"
method_name: "LindormVector"
authors: [Yan Wang, Jian Zhou, Sai Huang, Chao Dou, Hanwen Tian, Zhijie Jiang, Zongning Zhang, Xiaoqi Li, Zhencan Peng, Chunhui Shen, Wei Zhang, Feifei Li, Dong Deng]
year: 2026
venue: SIGMOD Companion
tags: [vector-database, approximate-nearest-neighbor, cloud-native, distributed-storage, ivfpq, product-quantization, hnsw, lindorm]
zotero_collection: SIGMOD/2026
image_source: local
created: 2026-07-09
---

# 论文笔记：LindormVector: A Distributed Vector Engine on a Cloud-Native Multi-Model NoSQL Database

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | Alibaba Cloud (阿里巴巴云), University of Science and Technology of China, Rutgers University |
| 日期 | May 2026 (SIGMOD Companion '26) |
| 项目主页 | — |
| 对比基线 | [[Milvus]], [[Elasticsearch]], [[PGVector]], [[Qdrant]] |
| 链接 | [DOI](https://doi.org/10.1145/3788853.3803088) |

---

## 一句话总结

> LindormVector 在阿里云多模型 NoSQL 数据库 Lindorm 上构建了分布式向量搜索引擎，采用 IVFPQ 索引 + 优化的 K-Means 聚类 + HNSW 质心搜索，在十亿级向量上实现毫秒级延迟和高召回率，显著超越专用和通用向量数据库。

---

## 核心贡献

1. **分布式向量引擎设计**: 在 Lindorm 的共享存储架构上原生构建向量引擎，将所有数据持久化到 [[LindormDFS]]，实现弹性扩展、容错和高可用，同时保持低成本
2. **超大 nlist 的 IVFPQ 索引**: 发现最优聚类数远大于传统 $\sqrt{n}$ 启发式，提出基于成本模型的 nlist 选择方法，并通过 [[ANNS|ANN 加速 K-Means]] 和 [[HNSW|优化质心搜索]] 支持数十万至百万级聚类
3. **生产级功能**: 支持实时索引更新、[[Hybrid Search|混合检索]] 和 [[Filtered Search|过滤检索]]，已在阿里云生产环境部署，最大集群管理超 1000 亿向量

---

## 问题背景

### 要解决的问题

在大模型和生成 AI 时代，企业需要对海量非结构化数据（文本、图像、音频、视频）进行高效语义检索和 [[RAG|检索增强生成]]。现有向量数据库面临两类系统的固有权衡：
- **专用向量数据库**（如 [[Milvus]]、Pinecone、Weaviate、[[Qdrant]]）：低延迟内存搜索，但难以扩展和与结构化数据集成
- **通用数据库**（如 [[PGVector]]、[[Elasticsearch]]、GaussDB-Vector）：提供一致性和容错，但查询性能受限

### 现有方法的局限

| 局限 | 涉及系统 | 说明 |
|------|----------|------|
| 查询延迟高 | Milvus、ES、Qdrant、SingleStore-V | 每个段（segment）都必须搜索，广播到所有段后合并 |
| 数据新鲜度低 | LSM-style 系统 | 新插入向量不能立即检索，需等待段合并 |
| 段合并开销 | Milvus、ES 等 | 后台段合并导致 I/O 和 CPU 峰值，延迟不可预测 |
| 负载不均衡 | GaussDB-Vector | 每个聚类独占一个节点，热点聚类成为瓶颈 |
| ACID 开销 | PGVector、GaussDB-Vector、SingleStore-V | 事务锁和验证开销增加查询延迟 |
| 随机 I/O | CosmosDB (DiskANN) | 图遍历需大量随机磁盘 I/O，更新处理差 |
| 网络瓶颈 | Harmony | 子向量分布到多服务器，引入过多网络通信 |

### 本文的动机

Lindorm 作为多模型云原生 NoSQL 数据库，具备低成本、容错、水平扩展等优势。LindormVector 的设计目标是在 Lindorm 的共享存储架构上实现高性能分布式向量搜索，同时保持 Lindorm 的核心优势。关键挑战在于找到能无缝集成到 Lindorm 架构的 [[ANNS]] 技术，并组合它们使性能达到甚至超过专用向量数据库。

---

## 方法详解

### 系统架构

LindormVector 采用 **共享存储架构**，作为 Lindorm 的原生引擎实现，与 [[LindormTable]] 宽表引擎紧密集成：

- **存储层**: 所有数据（包括向量索引）持久化在 [[LindormDFS]] 中，实现存储与计算分离
- **计算层**: 由 LDServer 节点组成，LDMaster 协调元数据、负载均衡和故障恢复
- **分区**: 数据按行键的一致性哈希分区为 "region"，每个 region 由一个 LDServer 逻辑管理
- **全局共享质心**: 所有 region 使用同一组聚类质心，确保量化向量跨 region 完美对齐

### 核心模块

#### 模块1: IVFPQ 索引设计

**设计动机**: 在共享存储系统中，最小化 I/O 操作、网络通信和内存占用是关键

**为什么选择 IVFPQ 而非 DiskANN**:
- [[DiskANN]] 在图遍历中需要反复从 LindormTable 获取全精度向量 → 多轮 get 操作
- [[IVFPQ]] 几乎所有计算在内存中的量化向量上完成，仅在最终重排阶段一次性获取全精度向量
- LindormTable 的批量 get 操作可高效支持一次性获取，大幅减少网络开销

**具体实现**:
- 使用 [[Product Quantization|PQ]] 或 [[RaBitQ|Binary Quantization]] 压缩全精度向量
- 使用 [[SIMD]] 加速量化向量扫描（参考 [[PQFastScan]]）
- 所有量化向量驻留内存，无需 get 操作即可快速距离计算

#### 模块2: 超大 nlist 优化

**关键观察**: 最优聚类数 $nlist$ 远大于传统 $\sqrt{n}$ 启发式，尤其当 $n$ 很大时

**原因**: 增加聚类数产生更细粒度的簇，大幅减少需要扫描的量化向量数量，在不牺牲召回率的情况下提高查询效率

**挑战与解决方案**:
- 如何高效训练数十万质心的 K-Means → [[ANNS|ANN 加速 K-Means]]
- 如何加速查询时的质心搜索 → [[HNSW]] 质心索引 + 优化贪心遍历

#### 模块3: 索引持久化

向量索引持久化在 LindormTable 的三张宽表中：

| 宽表 | 存储内容 |
|------|----------|
| 表1 | 所有质心元数据，包括用于快速质心搜索的 HNSW 图 |
| 表2 | 量化元数据，如 PQ 码本 |
| 表3 | 量化向量本身（按 region 水平分区） |

系统启动时，每个 LDServer 将共享质心、量化元数据和其 region 中的量化向量加载到内存。

#### 模块4: 实时索引更新

- **插入**: 新向量按路由键分配到 region → 分配到最近质心 → 量化后追加到对应 posting list
- **删除**: 目标向量与 posting list 末尾元素交换以保持紧凑性 → 长度递减
- **持久化**: [[WAL]] 保证 LindormTable 和 LindormVector 均持久化后才确认提交

#### 模块5: 混合检索与过滤检索

- **混合检索**: [[LindormVector]] 基于向量相似度检索 + [[LindormSearch]] 基于词法匹配（如 [[BM25]]）检索 → 使用 [[RRF|Reciprocal Rank Fusion]] 融合
- **过滤检索**: 基于谓词选择性动态选择策略：
  - **预过滤**（高选择性）：先评估谓词，再对合格行做向量比较
  - **后过滤**（低选择性）：先做向量搜索，再在协调器评估谓词；不足时增大 $nprobe$ 重试
  - **属性下推**（Attribute Pushdown）：将结构化属性嵌入向量索引的列式文件中，扫描时先评估谓词再做距离计算，避免昂贵的行获取

---

## 关键公式

### 公式1: K-Means 每轮迭代复杂度（暴力法）

$$
T_{\text{flat}} = O(m \cdot d \times nlist)
$$

**含义**: 暴力 K-Means 每轮需计算 $m$ 个训练向量到 $nlist$ 个质心的距离，每个距离计算复杂度 $O(d)$

**符号说明**:
- $m$: 训练样本数（通常比 $nlist$ 大 1-2 个数量级）
- $d$: 向量维度（通常 128-4096）
- $nlist$: 聚类数（通常数十万至百万）

### 公式2: ANN 加速 K-Means 复杂度

$$
T_{\text{anns}} = O(m \cdot \log nlist)
$$

**含义**: 使用 HNSW 索引替代暴力搜索，每个向量通过次线性搜索找到最近质心

**符号说明**:
- $m$: 训练样本数
- $nlist$: 聚类数
- $\log nlist$: HNSW 贪心搜索复杂度

### 公式3: 成本模型总搜索时间

$$
T_{\text{total}} = T_1 + T_2
$$

**含义**: 最优 $nlist$ 通过最小化质心搜索时间 $T_1$ 和码扫描时间 $T_2$ 之和来选择

**符号说明**:
- $T_1$: 质心搜索时间（使用估计的 $nprobe$ 执行实际质心搜索的平均时间）
- $T_2$: 码扫描时间（估计需扫描的量化向量数 $\times$ 每 SIMD 批次平均处理时间）

### 公式4: recall@k 定义

$$
\text{recall@}k = \frac{|\text{retrieved } k \cap \text{true } k\text{-NN}|}{k}
$$

**含义**: 检索到的 $k$ 个向量中包含真实最近邻居的比例

### 公式5: PQ 编码

$$
\text{PQ}(\mathbf{v}) = (c_1, c_2, \ldots, c_m), \quad c_i = \arg\min_{j \in \{1, \ldots, 2^b\}} \|\mathbf{v}_i - \boldsymbol{\mu}_{i,j}\|
$$

**含义**: 将 $d$ 维向量拆分为 $m$ 个子空间，每个子向量编码为其最近质心的索引

**符号说明**:
- $\mathbf{v}_i$: 第 $i$ 个子向量
- $\boldsymbol{\mu}_{i,j}$: 第 $i$ 个子空间的第 $j$ 个质心
- $c_i$: 第 $i$ 个子空间的编码（$b$ bits）
- 总存储: $m \times b$ bits 每向量

### 公式6: RaBitQ 二值量化

$$
\text{RaBitQ}(\mathbf{v}) = \text{sign}(\mathbf{R} \cdot \mathbf{v})
$$

**含义**: 对向量空间施加固定随机正交旋转 $\mathbf{R}$，然后取每个旋转后坐标的符号，生成 $d$-bit 二值码

**符号说明**:
- $\mathbf{R}$: 随机正交旋转矩阵
- $\text{sign}(\cdot)$: 符号函数

### 公式7: PCA 压缩质心

$$
\mathbf{c}' = \mathbf{W}_{\text{PCA}}^T \mathbf{c}, \quad \text{retaining components explaining 95\% variance}
$$

**含义**: 对全精度质心应用 [[PCA]]，仅保留解释 95% 方差的主成分，在图遍历中使用压缩质心以减少内存带宽

---

## 关键图表

### Figure 1: Lindorm 系统概览

![[LindormVector_fig1_overview.png]]

**说明**: Lindorm 整体架构。底层是 LindormDFS 分布式文件系统，上层集成了宽表、搜索、列存、向量、时序、计算和 AI 引擎，通过 LindormSQL 统一查询接口访问。

### Figure 2: LindormVector 架构

![[LindormVector_fig2_architecture.png]]

**说明**: LindormVector 的架构。数据按 region 分区在 LDServer 上，内存中包含质心索引、posting list 和量化向量，全精度向量持久化在 LindormDFS。LDMaster 协调节点管理。

### Figure 3: Code Scan Time vs nlist

![[LindormVector_fig3_code_scan_time.png]]

**说明**: 在 SIFT100M 和 DEEP100M 上，随着 $nlist$ 增大，码扫描时间快速下降后趋于平缓。每个 $nlist$ 选择达到 0.90 recall@10 的最小 $nprobe$。PQ 配置 $m=d$, $b=4$。

### Figure 4: 聚类粒度对码扫描的影响

![[LindormVector_fig4_cluster_granularity.png]]

**说明**: 聚类粒度（与 $nlist$ 成正比）对码扫描效率的影响。粗聚类（左）需扫描大量量化向量；细聚类（中/右）查询区域更局部化，扫描量大幅减少。

### Figure 5: 不同 nlist 下的 QPS-Recall 曲线

![[LindormVector_fig5_qps_recall_nlist.png]]

**说明**: SIFT100M 和 DEEP100M 上不同 $nlist$ 的 QPS-recall 曲线。$nlist = \sqrt{n} = 10K$ 并非最优，$nlist = 200K$ 在两个数据集上均取得最佳性能。

### Figure 6: 成本模型 nlist 估计

![[LindormVector_fig6_nlist_estimation.png]]

**说明**: 成本模型在不同 recall@topk 目标下对 $nlist$ 的估计。估计延迟曲线（Est.）紧密跟踪实测趋势（Real），在所有配置下均选择接近最优的 $nlist$。

### Figure 7: K-Means 聚类时间

![[LindormVector_fig7_kmeans_time.png]]

**说明**: 四种 K-Means 算法在 1000 万向量上的聚类时间。HNSW 比 FLAT 减少 68%-91%；LindormVector 优化（LV）在 HNSW 基础上再提升 40%-81%；LV+ 入口点优化再提升 7%-15%。

### Figure 8: 优化质心搜索评估

![[LindormVector_fig8_centroid_search.png]]

**说明**: 500K 质心上 $k$-最近质心搜索的 QPS-recall 曲线。Lindorm HNSW 在所有 $k$ 值下均优于 Vanilla HNSW：$k=10$ 提升 32%-154%，$k=100$ 提升 25%-165%，$k=1000$ 提升 68%-349%。

### Figure 9: 与基线系统 QPS-Recall 对比

![[LindormVector_fig9_qps_recall_comparison.png]]

**说明**: LindormVector 与 Milvus、Elasticsearch、PGVector、Qdrant 在四个数据集上的 QPS vs Recall@10 曲线。LindormVector 在相同召回率下提供 1-2 个数量级更高的 QPS，在 Cohere-10M 上超越 PGVector 和 Milvus 超 10 倍。

### Figure 10: 可扩展性

![[LindormVector_fig10_scalability.png]]

**说明**: Synthetic-1B 数据集上，从 1 到 10 个 LDServer 节点的客户端 QPS。每节点数据量固定为 $10^8$，总数据量从 $10^8$ 增至 $10^9$。QPS 随节点增加保持稳定甚至上升（$nprobe$ 较小时）。

### Table 1: 评估数据集汇总

| 规模 | 名称 | 向量数 | 维度 $d$ | 原始大小 |
|------|------|--------|----------|----------|
| Medium | Cohere-10M | $10^7$ | 768 | 30GB |
| Medium | BioASQ-10M | $10^7$ | 1024 | 40GB |
| Large | Turing-1B | $10^9$ | 100 | 400GB |
| Large | SpaceV-1B | $10^9$ | 100 | 400GB |
| Ultra-Large | Synthetic-1B | $10^9$ | 1024 | 4TB |

### Table 2: K-Means 均方误差对比

| 数据集 | BioASQ-10M | BioASQ-10M | Cohere-10M | Cohere-10M |
|--------|------------|------------|------------|------------|
| nlist | 100K | 500K | 100K | 500K |
| FLAT | 3.474×10¹¹ | 2.908×10¹¹ | 1.823×10⁶ | 1.518×10⁶ |
| HNSW | 3.474×10¹¹ | 2.917×10¹¹ | 1.829×10⁶ | 1.549×10⁶ |
| LV | 3.475×10¹¹ | 2.908×10¹¹ | 1.856×10⁶ | 1.521×10⁶ |
| LV+ | 3.474×10¹¹ | 2.906×10¹¹ | 1.855×10⁶ | 1.515×10⁶ |

**关键发现**: 所有 ANNS 加速方法的 MSE 与暴力 FLAT 几乎完全一致，仅微小波动，证明聚类质量无损。

### Table 3: 索引构建时间

| 系统 | Cohere-10M | BioASQ-10M | Turing-1B | SpaceV-1B |
|------|------------|------------|-----------|-----------|
| Milvus | 12454s | 6270s | 16h | 17.5h |
| Qdrant | 1980s | 2750s | / | / |
| PGVector | 1165s | 1843s | 3.33h | 3.3h |
| Elasticsearch | 4776s | 3798s | 19.6h | 9.4h |
| **LindormVector** | **902s** | **990s** | **16h** | **12.5h** |

**关键发现**: 中等规模数据集上 LindormVector 比现有系统快数倍（BioASQ-10M 上比 Milvus 快 6 倍）；十亿规模上 PGVector 最快但因其 $nlist$ 极小（10K）牺牲查询质量。

### Table 4: 过滤检索性能 (Cohere-10M, nlist=50000)

| 选择性 (%) | Pre-filter L | Pre-filter R | Post-filter L | Post-filter R | Pushdown L | Pushdown R |
|------------|-------------|-------------|--------------|--------------|-----------|-----------|
| 1 | 734.1 | 296.4 | — | — | 10.60 | 95.0 |
| 10 | — | — | 33.46 | 70.6 | 9.38 | 92.7 |
| 50 | — | — | 33.19 | 74.3 | 11.78 | 95.8 |
| 90 | — | — | 38.64 | 75.9 | 12.81 | 96.2 |
| 99 | — | — | 15.80 | 95.2 | 10.60 | 95.4 |

**关键发现**: 属性下推是唯一在全选择性范围内有效的方法，保持约 10-13ms 延迟和约 95% 召回率；预过滤仅在高选择性（1%）时可用但延迟高（734ms）；后过滤在中选择性时召回率降至 70%-75%。

---

## 实验

### 数据集

| 数据集 | 规模 | 特点 | 用途 |
|--------|------|------|------|
| Cohere-10M | $10^7$, 768维 | Wikipedia 多语言嵌入 | 中等规模评估 |
| BioASQ-10M | $10^7$, 1024维 | 生物医学语义索引 | 中等规模评估 |
| Turing-1B | $10^9$, 100维 | 微软大规模网页搜索 | 大规模评估 |
| SpaceV-1B | $10^9$, 100维 | 微软 Bing 用户意图 | 大规模评估 |
| Synthetic-1B | $10^9$, 1024维 | BioASQ+高斯噪声合成 | 可扩展性评估 |
| SIFT100M | $10^8$, 128维 | 经典 ANN benchmark | nlist 分析 |
| DEEP100M | $10^8$, 96维 | 深度描述符 | nlist 分析 |

### 实现细节

- **硬件**: Alibaba Cloud ECS `ecs.g9i.8xlarge`（3.2 GHz Intel Xeon Granite Rapids, 32 vCPUs, 128GB 内存），ESSD PL1 块存储
- **PQ 配置**: 中等规模 $m=d, b=4$；十亿规模 $m=d/2, b=4$
- **nlist**: 中等规模 50,000；十亿规模 200,000
- **PCA 降维**: BioASQ 1024→384，Cohere 768→320（保留 95% 方差）
- **重排因子**: 3（中等规模），2（可扩展性实验）

### 关键实验结果

1. **ANN 加速 K-Means**: 聚类时间减少 5-10 倍，质量几乎无损
2. **优化质心搜索**: 相比 Vanilla HNSW，QPS 提升 25%-349%（取决于 $k$ 值）
3. **查询性能**: 相同召回率下 QPS 领先 1-2 个数量级
4. **可扩展性**: 1→10 节点 QPS 保持稳定甚至上升
5. **实时更新**: 中等规模 3-4ms，十亿规模 33-34ms 写延迟
6. **多租户**: 每租户仅 0.35MB 索引内存，3.11ms 延迟，recall@10 = 0.99

---

## 批判性思考

### 优点

1. **架构设计优雅**: 全局共享质心 + posting list 对齐使 region 管理简化、无缝扩缩容、缓解热点
2. **成本模型实用**: 通过网格搜索少量候选 $nlist$，估计 $T_1 + T_2$ 选择最优值，经验证可靠
3. **ANN 加速 K-Means 创新**: 用 HNSW 替代暴力质心分配，将 $O(m \times nlist)$ 降至 $O(m \log nlist)$，支持百万级聚类
4. **生产验证**: 已在阿里云部署，最大集群管理 1000 亿+ 向量，不只是学术 demo
5. **全功能**: 实时更新、混合检索、过滤检索、属性下推、多租户、索引元数据共享等生产级功能齐全

### 局限性

1. **共享质心限制**: 全局共享一组质心假设所有数据可被统一聚类，若数据分布随时间漂移（concept drift），可能需要重建索引
2. **PQ 压缩精度损失**: 对高维数据（如 1024 维 Synthetic-1B），PQ 不可避免地引入近似误差，依赖最终重排弥补
3. **无 ACID 保证**: Lindorm 放弃强事务以保证效率，对需要严格一致性的场景不适用
4. **索引构建时间**: 十亿规模上（Turing-1B 16h）仍较长，PGVector 因用极小 $nlist$ 反而更快
5. **评估局限**: 基线系统在十亿规模上多数无法完成构建（Qdrant 超时、Milvus 内存不足），对比不完全

### 潜在改进方向

1. **增量 K-Means 更新**: 支持质心的增量更新而非全量重建，适应数据分布漂移
2. **学习量化方案**: 用 [[RaBitQ]] 的理论误差界指导量化方案选择，提供精度保证
3. **GPU 加速**: 利用 GPU 加速 PQ 距离计算和 SIMD 批扫描
4. **自适应 nprobe**: 根据查询分布和数据分布动态调整 $nprobe$

### 可复现性评估

- [ ] 代码开源
- [ ] 预训练模型
- [x] 训练细节完整（PQ 配置、nlist、nprobe、重排因子等）
- [x] 数据集可获取（均为公开 benchmark 数据集）

---

## 关联笔记

### 基于
- [[IVFPQ]]: LindormVector 的核心索引设计基础
- [[Product Quantization]]: 向量压缩的核心技术
- [[HNSW]]: 质心索引和 ANN 加速 K-Means 的核心数据结构

### 对比
- [[Milvus]]: 专用向量数据库，LSM-style 段管理，DiskANN 索引
- [[Elasticsearch]]: 通用搜索引擎，DiskBBQ 磁盘索引
- [[PGVector]]: PostgreSQL 扩展，IVFFLAT 索引，受 ACID 开销限制
- [[Qdrant]]: 专用向量数据库，仅 HNSW 索引
- [[DiskANN]]: 磁盘向量索引，随机 I/O 问题
- [[Harmony]]: 分布式向量数据库，子向量分布引入网络瓶颈

### 方法相关
- [[RaBitQ]]: 二值量化方法，LindormVector 的可选压缩方案
- [[K-Means Clustering]]: 聚类算法，LindormVector 用 ANN 加速
- [[SIMD]]: 单指令多数据，加速 PQ 码扫描
- [[PCA]]: 主成分分析，压缩质心用于图遍历
- [[LindormDFS]]: Lindorm 分布式文件系统
- [[LindormTable]]: Lindorm 宽表引擎
- [[WAL]]: 预写日志，保证一致性

### 硬件/数据相关
- [[SIFT100M]]: 经典 ANN benchmark 数据集
- [[DEEP100M]]: 深度描述符 benchmark 数据集

---

## 速查卡片

> [!summary] LindormVector
> - **核心**: 在 Lindorm 共享存储架构上构建 IVFPQ 向量引擎，全局共享质心 + 内存驻留量化向量
> - **方法**: 超大 nlist（远超 √n）+ ANN 加速 K-Means + HNSW 优化质心搜索 + PCA 压缩遍历
> - **结果**: 相同召回率下 QPS 领先 1-2 个数量级，毫秒级延迟，十亿级向量，生产部署 1000 亿+
> - **代码**: 未开源

---

*笔记创建时间: 2026-07-09*
