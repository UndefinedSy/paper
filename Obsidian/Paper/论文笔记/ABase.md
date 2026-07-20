---
title: "ABase: the Multi-Tenant NoSQL Serverless Database for Diverse and Dynamic Workloads in Large-scale Cloud Environments"
method_name: "ABase"
authors: [Rong Kang, Yanbin Chen, Ye Liu, Fuxin Jiang, Qingshuo Li, Miao Ma, Jian Liu, Guangliang Zhao, Tieying Zhang, Jianjun Chen, Lei Zhang]
year: 2025
venue: SIGMOD-Companion
tags: [multi-tenant, serverless, nosql, key-value-store, cloud-database, autoscaling, performance-isolation]
zotero_collection: _待整理
image_source: local
arxiv_html: https://arxiv.org/html/2505.07692
created: 2026-07-20
---

# 论文笔记：ABase

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | ByteDance Inc. |
| 日期 | June 2025 |
| 项目主页 | - |
| 对比基线 | [[DynamoDB]], [[Cassandra]] |
| 链接 | [arXiv](https://arxiv.org/abs/2505.07692) / [DOI](https://doi.org/10.1145/3722212.3724426) |

---

## 一句话总结

> ABase 是字节跳动自研的多租户 NoSQL Serverless 数据库，通过缓存感知隔离、分层缓存、预测式弹性伸缩和多资源重调度四大机制，支撑了 13 亿峰值 QPS 和 1 EB 存储规模的生产环境。

---

## 核心贡献

1. **Cache-Aware Isolation Mechanism**: 将缓存命中率纳入 [[请求单元 (RU)|RU]] 计算，设计代理层+分区层的两级流量管控和 Dual-Layer [[WFQ|加权公平队列]]，实现缓存感知的精确性能隔离
2. **Hierarchical Caching Mechanism**: 代理层基于 [[AC-LRU|Active-Update LRU]] + [[Limited Fan-out Hash Routing|限扇出哈希路由]]解决热键问题；DataNode 层基于 [[SA-LRU|Size-Aware LRU]] 优化多尺寸 KV 的缓存效率
3. **Predictive Autoscaling Policy**: 集成 [[Prophet]] 模型与历史均值的集成预测方案，自动处理突发流量、多周期性和趋势变化
4. **Multi-Resource Rescheduling Algorithm**: 基于 L2-Norm Loss 的启发式多资源重调度，同时均衡 RU 和存储两个维度的 DataNode 负载
5. **Production Analysis**: 覆盖 13 亿峰值 QPS、1 EB 存储的线上验证，降低 65% 弹性扩容告警

---

## 问题背景

### 要解决的问题

大规模云环境中，多租户 NoSQL Serverless 数据库面临三大挑战：
1. **缓存对性能隔离的影响**：代理缓存命中时请求不进 DataNode，DataNode 缓存命中时无磁盘 IO，传统 RU 计算无法反映实际资源消耗，导致流量管控不准确
2. **流量动态变化**：租户流量趋势变化导致配额耗尽或浪费；访问分布变化引发热键压力或缓存命中率骤降
3. **DataNode 负载不均衡**：不同租户对 CPU 和磁盘的需求差异悬殊（有的吞吐高存储低，有的反之），粗粒度的数据布局导致资源利用率低下

### 现有方法的局限

- **DynamoDB**：探索了多租户流量管控和负载均衡的必要性，但未披露弹性算法和重调度细节；未讨论缓存对隔离机制的影响
- **CosmosDB**：单库 100 万 RU/s 上限，难以支持字节跳动级别的超大单租户
- **Firestore**：专注 Web/Mobile 开发者场景，生态绑定，缺乏超大规模多租户优化
- 传统 NoSQL（Cassandra 等）：为单租户/本地化静态资源分配设计，缺乏弹性伸缩和细粒度隔离原语

### 本文的动机

字节跳动运营抖音社交媒体、电商、搜索、广告、推荐、LLM KV Cache 等极多样化业务，单业务线峰值 QPS 达 4.5 亿，存储超 11 PB；同时这些业务负载具有高动态性（双十一活动等）。现有系统无法满足如此规模和多样性，需从零设计一套多租户 Serverless 数据库。

---

## 方法详解

### 系统架构概览

![[ABase_fig1_architecture_p5.png]]

**说明**：ABase 多租户架构由三层组成：

- **Control Plane（控制面）**: [[Meta Server|MetaServer]]（全局元数据、DataNode 健康监控、分区迁移）+ [[Autoscaler|自动扩缩器]]（基于时序预测的 RU/Storage 弹性伸缩）+ [[Rescheduler|重调度器]]（负载均衡迁移）
- **Data Plane（数据面）**: 多个 Resource Pool，每个 Pool 含若干 DataNode；DataNode 内嵌 [[SA-LRU]] Cache + Dual-Layer [[WFQ]]；各分区有独立 `partition_quota`
- **Proxy Plane（代理面）**: 每个租户独立的 Proxy Group，内嵌 [[AC-LRU]] Cache，负责路由、代理层流量管控（`proxy_quota`）、热键防护

数据模型：[[Resource Pool]] → Tenant → Table → Item（Key），数据按 Hash 分区，跨 AZ 多副本。

### 核心模块

#### 模块1：Cache-Aware RU 计算

**设计动机**：缓存命中改变请求资源消耗路径，必须将缓存命中率纳入 [[请求单元 (RU)]] 估算，否则隔离机制失效。

**具体实现**：
- **写操作**：$RU_{write} = S_{write} / U$（$U = 2\text{KB}$），含复制则乘以副本数 $r$
- **读操作**：$RU_{read} = \mathbb{E}[S_{read}] \times (1 - \mathbb{E}[R_{hit}]) / U$，用最近 $k$ 次请求的移动平均估算期望大小和命中率；代理缓存命中则不计费
- **复杂操作**（如 HGetAll）：分解为 HLen + Scan 两阶段分别计算

#### 模块2：Hierarchical Request Restriction（两级流量管控）

![[ABase_fig2_isolation_p7.png]]

**说明**：Cache-Aware 性能隔离架构，展示了代理层/分区层两级限流 + Dual-Layer WFQ 的全链路。

**代理层（proxy-level）**：
- 每个代理分得 `proxy_quota = tenant_quota / proxy数量`，允许临时超量 2x
- MetaServer 异步监控并在超量时将代理限流回标准值（避免中心化实时交互的延迟）
- 代理层拦截超配流量，防止突发流量冲击 DataNode 影响其他租户

**分区层（partition-level）**：
- `partition_quota = tenant_quota / partition数量`，单分区最多超量 3x
- 超过即在 DataNode 入队处直接拒绝，避免 DataNode 浪费资源处理无效请求

#### 模块3：Dual-Layer WFQ

**设计动机**：DataNode 上共存多租户分区，需防止不同请求类型（大/小、读/写）的互相干扰，参考 [[2DFQ]] 思想。

**具体实现**：
- 将请求按类型（读/写）× 大小（大/小）分入 4 组独立的双层 WFQ
- 上层 **CPU-WFQ**：处理请求，命中缓存直接返回；未命中则进入下层
- 下层 **I/O-WFQ**：处理磁盘 IO，含 basic threads + extra threads（防单租户垄断线程池）
- WFQ 以最小 [[Virtual Finish Time (VFT)|虚拟完成时间]] 为优先级，按分区配额比例加权

**4 条工程优化规则**：
- Rule 1: CPU-WFQ 按 RU 计 cost，IO-WFQ 按 IOPS 计 cost（I/O 耗时相近）
- Rule 2: 读写并发限制 + 写操作 RU 上限（防 LavaStore compaction 引起抖动）
- Rule 3: 单租户最多占 90% CPU-WFQ 资源（防突发导致其他租户饥饿）
- Rule 4: basic threads 被单租户垄断时，自动新增 extra threads 服务其他租户

#### 模块4：Dual-Layer Caching

**代理层缓存（AC-LRU + Limited Fan-out Hash）**：
- [[AC-LRU|Active-Update LRU]]：临近过期时主动刷新热键，防止缓存到期引发的请求尖峰
- [[Limited Fan-out Hash Routing|限扇出哈希路由]]：$N$ 个代理分为 $n$ 组，每个请求哈希到一个 ProxyGroup，再随机选一个代理
  - 大 $n$：每个代理接收 $1/n$ 流量，缓存命中率更高
  - 小 $n$：热键时流量分散到更多代理（$= N/n$ 个），降低热键压力
  - 可动态调节 $n$ 平衡两者 trade-off

**DataNode 层缓存（SA-LRU）**：
- [[SA-LRU|Size-Aware LRU]]：相比标准 LRU，优先驱逐"占内存多但命中少"的大 KV
- 保留小尺寸 KV（访问代价低），提升整体缓存命中率和内存利用率

#### 模块5：Predictive AutoScaling

**设计动机**：传统预测方案难以处理 ABase 的三个实际挑战：
- Issue 1: 偶发突发 + 分区迁移时的指标噪声（噪声和真实突发难以区分）
- Issue 2: 多样周期（除日/周外还有 3.5 天等非标准周期）+ 趋势突变
- Issue 3: 每日不固定时间的稳定非周期性突发（不能当 outlier 过滤）

**集成预测方案**：

预处理：
- 多指标协同去噪：Usage 和 Quota 同时出现尖峰 → 判定为噪声过滤
- 启发式过滤：10 天内仅出现 1 次的峰值判定为偶发，不触发扩容
- [[变点检测|Change Point Detection]]：识别趋势转折，让模型更关注近期数据

预测阶段（集成）：
- [[PSD|Power Spectral Density]] 分析确定时序周期
- [[Prophet]] 模型：擅长有明显趋势和周期的序列
- [[历史均值法]]：趋势平稳时稳定性更好
- 加权集成两者；若预测值显著低于历史数据（Issue 3），直接用最近一个周期的历史值

扩缩策略（Algorithm 1）：
- 预测未来 7 天最大用量 $U_{max}$
- $U_{max} > 0.85 \times Q_T$ → 扩容，令 $Q_T \leftarrow U_{max}/0.65$
- $U_{max} < 0.65 \times Q_T$ 且 7 天内未扩缩 → 缩容
- 扩容后若 $Q_P > UP$ → 触发分区分裂

#### 模块6：Multi-Resource Rescheduling（Intra/Inter-Pool）

**设计动机**：不同租户 RU/Storage 资源需求差异大，简单布局导致 DataNode 负载严重不均（有的 CPU 高存储低，有的反之）。

**核心定义**：

**Load Indicator（负载向量）**：
- 副本负载向量 $RE^{ld} = (RE^{ld}_1, \cdots, RE^{ld}_{24})$（过去7天，每小时取最大值，去掉绝对时间）
- DataNode 负载 = 其上所有副本负载向量之和，再取全天最大值：

$$
DN^{ld}(RP^{ld}) = \max_i \left( \sum_{RE \in DN(RP)} RE^{ld}_i \right) \quad \text{for } i \in \{1,2,...,24\}
$$

**Optimal Load（最优负载比例）**：资源池级别 RU/存储的平均利用率比例：

$$
\langle R, S \rangle = \left( \frac{RP^{ld}_{ru}}{RP^{cap}_{ru}}, \frac{RP^{ld}_{sto}}{RP^{cap}_{sto}} \right)
$$

**Migration Gain（迁移收益）**：用 L2-Norm 衡量 DataNode 偏离最优负载的程度：

$$
\mathcal{L}(DN) = \sqrt{\left(\frac{DN^{ld}_{ru}}{DN^{cap}_{ru}} - R\right)^2 + \left(\frac{DN^{ld}_{sto}}{DN^{cap}_{sto}} - S\right)^2}
$$

迁移增益为迁移前后两节点最大偏差的减少量：

$$
\mathcal{G}(RE, Des\_DN) = \max[\mathcal{L}(RE.DN), \mathcal{L}(Des\_DN)] - \max[\mathcal{L}(RE.DN.Remove(RE)), \mathcal{L}(Des\_DN.Add(RE))]
$$

**DataNode 分组**（以 RU 为例，阈值 $\theta$）：
- $\mathcal{S}_L$（低负载）：$DN^{ld}_{ru}/DN^{cap}_{ru} \leq R - \theta$
- $\mathcal{S}_M$（中等负载）：$R - \theta < DN^{ld}_{ru}/DN^{cap}_{ru} \leq R$
- $\mathcal{S}_H$（高负载）：其他

**Intra-Pool 算法（Algorithm 2）**：从 $\mathcal{S}_H$ 中的 DataNode 找收益最大的副本，迁移到 $\mathcal{S}_L$ 中满足放置约束的目标节点，每 10 分钟执行一次。

**Inter-Pool 算法**：从低负载池腾空部分低利用率 DataNode，重分配给高负载池，再在两池内分别调用 Intra-Pool 算法均衡。

---

## 关键公式

### 公式1: [[请求单元 (RU)|Write RU 计算]]

$$
RU_{write} = S_{write} / U, \quad \text{总费用} = r \cdot RU_{write}
$$

**含义**：写操作 RU 正比于写入值大小，$U=2\text{KB}$；含 $r$ 副本时费用乘以 $r$

**符号说明**：
- $S_{write}$：写入值大小（字节）
- $U$：单位大小，经验设为 2KB
- $r$：副本数

### 公式2: [[请求单元 (RU)|Read RU 估算]]

$$
RU_{read} = \mathbb{E}[S_{read}] \times (1 - \mathbb{E}[R_{hit}]) / U
$$

**含义**：读操作 RU 结合了期望返回大小和缓存未命中率，命中缓存的部分不消耗 RU

**符号说明**：
- $\mathbb{E}[S_{read}]$：最近 $k$ 次请求的移动平均读取大小
- $\mathbb{E}[R_{hit}]$：最近 $k$ 次请求的缓存命中率移动均值
- $U$：单位大小 2KB

### 公式3: [[Virtual Finish Time (VFT)|WFQ 虚拟完成时间]]

$$
\text{wReqCost}(Q_i) = \frac{\text{Cost}(Q_i)}{Q_i / \sum Q_p}
$$

$$
\text{VFT}(Q_i) = \text{preVFT}_{T_i} + \text{wReqCost}(Q_i)
$$

**含义**：按分区配额比例对请求代价加权，VFT 在同租户请求间累积，防止高配额租户始终优先

**符号说明**：
- $Q_i$：分区 $i$ 的配额
- $\sum Q_p$：DataNode 上所有分区的配额之和
- $\text{preVFT}_{T_i}$：该租户上次请求的 VFT（累积）
- $\text{Cost}(Q_i)$：CPU-WFQ 按 RU 计，IO-WFQ 按 IOPS 计

### 公式4: [[DataNode 负载向量|负载向量]]

$$
DN^{ld}(RP^{ld}) = \max_i \left( \sum_{RE \in DN(RP)} RE^{ld}_i \right) \quad \text{for } i \in \{1,2,...,24\}
$$

**含义**：DataNode 负载 = 所有副本负载向量按小时叠加后取全天峰值，消除日内绝对时间差异

**符号说明**：
- $RE^{ld}_i$：副本在第 $i$ 小时的负载（过去7天最大值）
- $i$：小时索引，1..24

### 公式5: [[迁移收益函数|Migration Gain]]

$$
\mathcal{L}(DN) = \sqrt{\left(\frac{DN^{ld}_{ru}}{DN^{cap}_{ru}} - R\right)^2 + \left(\frac{DN^{ld}_{sto}}{DN^{cap}_{sto}} - S\right)^2}
$$

$$
\mathcal{G}(RE, Des\_DN) = \max[\mathcal{L}(RE.DN), \mathcal{L}(Des\_DN)] - \max[\mathcal{L}(RE.DN.Remove(RE)), \mathcal{L}(Des\_DN.Add(RE))]
$$

**含义**：L2-Norm 衡量 DataNode 多维度负载偏离最优点的距离；迁移收益为迁移前后最大偏差的减少量，正收益触发迁移

**符号说明**：
- $R, S$：资源池最优 RU/存储利用率比例
- $DN^{cap}_{ru}$, $DN^{cap}_{sto}$：DataNode 的 RU/存储容量

---

## 关键图表

### Figure 1: ABase 多租户架构

![[ABase_fig1_architecture_p5.png]]

**说明**：三平面架构全景图。Control Plane 含 MetaServer/Autoscaler/Rescheduler；Data Plane 含多个 Resource Pool，每个 Pool 有多个 DataNode（内含 SA-LRU Cache 和 Dual-Layer WFQ）；Proxy Plane 含各租户的 Proxy Group（内含 AC-LRU Cache 和 Limit Fan-out Hash Routing）。

### Figure 2: Cache-Aware 性能隔离

![[ABase_fig2_isolation_p7.png]]

**说明**：代理层（Proxy Quota 拦截超配流量）→ DataNode 请求队列（Partition Quota 拒绝超限请求）→ Dual-Layer WFQ（CPU-WFQ 命中缓存直返，未命中进 IO-WFQ）。4 类独立队列（Small/Large Read/Write）确保不同请求类型间延迟对齐。

### Figure 3 & 4: 生产统计数据

![[ABase_fig3_4_production_p9.png]]

**说明**：
- Figure 3: 资源池内租户按 RU/Storage/Read Ratio 分布，圆点颜色越深读比例越高，呈现极大的多样性
- Figure 4: 关键指标分位数（a）99th 延迟均低于 SLA 66% 以下（b）50% 租户缓存命中率 > 93.5%（c）读比例两极分化（d）KV 大小 P50=0.12KB，P99=308KB

### Figure 5: 双十一期间租户动态变化

![[ABase_fig5_dynamics_p10.png]]

**说明**：6 个租户在双十一期间的 QPS/缓存命中率/延迟变化案例。代表了"QPS 升缓存稳定""QPS 升缓存降""QPS 和缓存同升""QPS 稳缓存降""短时 QPS 暴升缓存骤降"等典型场景；资源池级别指标（Figure 5f）显示整体 QPS 和缓存率保持稳定。

### Figure 6 & 7 & 8: 实验验证

![[ABase_fig6_7_8_experiments_p11.png]]

**说明**：
- Figure 6: Proxy Quota 效果——租户1突发后未开代理限流时租户2 QPS 骤降至 0；开启后两者恢复正常
- Figure 7: Partition Quota + WFQ 效果——只有 WFQ 时租户1延迟 20x；开启分区限流后两者延迟均低
- Figure 8a: 预测式扩容案例，Day 10 提前扩容防止 Day 17 触发限流；Figure 8b: 部署自动扩缩后告警量下降 65%

### Figure 9 & 10: 重调度效果

![[ABase_fig9_10_rescheduling_p12.png]]

**说明**：
- Figure 9: 1000 节点资源池重调度前(a)后(b)对比：RU 标准差降低 74.5%，存储方差降低 84.8%
- Figure 10: 在线环境中重调度后 DataNode 最大 QPS 逐渐向均值收拢

### Table 1: ByteDance 典型业务负载特征

| 业务线 | 场景 | 标准化吞吐 | 标准化存储 | 缓存命中率 | 读比例 | KV 大小 | TTL |
|--------|------|-----------|-----------|-----------|--------|---------|-----|
| 社交(抖音) | 评论 | 250 | 125 | 54% | 100% | 0.1KB | - |
| 社交(抖音) | 私信 | 25 | 678 | 74% | 100% | 1KB | - |
| 电商 | 商品元数据 | 575 | 429 | 92% | 100% | 1KB | - |
| 搜索 | 正排数据 | 1500 | 639 | 99% | 100% | 1KB | - |
| 广告 | 消息 Joiner | 2750 | 938 | 18% | 25% | 10KB | 3小时 |
| 推荐 | 去重 | 5325 | 6257 | 76% | 50% | 2KB | 15天 |
| LLM | 远程 KV Cache | 10000 | 5760 | 0% | 85% | 5MB | 1天 |

**说明**：极大的多样性——LLM 缓存命中率为 0（绕过缓存直读底层日志），广告写重且命中率极低，搜索读重且命中率极高；KV 大小跨越 0.1KB 到 5MB，TTL 从 3 小时到不设 TTL

### Table 2: 代理缓存效果验证

| 租户 | Proxy 数 | 组数 | 命中率提升 | RU 节省 |
|------|---------|------|-----------|---------|
| 社交媒体 1 | 375 | 75 | 5% → 86% | 85% |
| 社交媒体 2 | 162 | 63 | 25% → 67% | 70% |
| 社交媒体 3 | 115 | 3015 | 10% → 33% | 38% |
| 电商 1 | 790 | 15 | 24% → 60% | 61% |
| 电商 2 | 1511 | 15 | 24% → 60% | 57% |
| 电商 3 | 4204 | 15 | 24% → 60% | 79% |

**说明**：仅通过调整路由策略（零数据迁移），命中率提升 23-81 个百分点，RU 节省 38-85%

---

## 实验

### 生产统计

- 资源池规模：1000+ DataNodes
- 整体指标：峰值 QPS 13 亿，总存储 1 EB；单租户最大 QPS 4.5 亿，最大存储 11 PB
- P99 延迟：所有租户均低于 SLA 的 66%，90% 租户低于 SLA 的 24%

### 资源利用率提升（对比 ABase-Pre 单租户版）

| 资源 | ABase-Pre（单租户） | ABase（多租户） |
|------|---------------------|----------------|
| CPU | 17% | 44% |
| Memory | 52% | 63% |
| Disk | 27% | 46% |

**说明**：单租户设计为容纳故障需预留大量空闲（至多使用 2/3 以防单节点故障 1.5x 流量）；多租户 N 节点冗余每节点只增加 1/N 负载，可维持更高整体利用率

### 弹性伸缩

- 部署自动扩缩后，扩容相关紧急告警（Oncall）数量下降约 **65%**
- 预测式扩容可在用量达到配额的 85% 前 7 天预判并扩容

### 重调度（1000 DataNode 离线实验）

- RU 标准差降低 **74.5%**
- 存储方差降低 **84.8%**
- 在线执行周期：每 10 分钟

---

## 批判性思考

### 优点

1. **工程细节丰富**：RU 计算将缓存命中率显式建模，解决了一个被其他系统忽视的现实问题
2. **多层次协同防御**：Proxy Quota → Partition Quota → Dual-Layer WFQ 三层递进，各层有明确分工
3. **负载向量设计精巧**：按小时维度聚合取每天最大值，既消除了绝对时间差异又保留了日内周期性规律
4. **生产级验证**：13亿 QPS、1EB 规模，且覆盖了多种极端场景（双十一、热键事件等）

### 局限性

1. **缓存一致性未深入讨论**：论文使用 eventual consistency，AC-LRU 的主动刷新在高更新频率下可能引入数据新鲜度问题，但未详述
2. **Limited Fan-out n 的选择缺乏自动化**：论文描述了"调整 n"但未给出自动调优算法，实际运维中需人工干预
3. **重调度对副本迁移代价建模简单**：算法只考虑负载均衡增益，未考虑迁移本身的 IO 代价和时间窗口影响
4. **预测方案依赖 30 天历史数据**：新租户或刚做业务调整的租户历史数据不足，预测可能不准

### 潜在改进方向

1. 自适应调整 Limited Fan-out 的 $n$ 值，基于实时热键检测动态分组
2. 结合 Cost-Benefit 分析对重调度决策做更细粒度的代价建模
3. 引入在线学习方法对新租户的 workload 做 cold-start 预测

### 可复现性评估

- [ ] 代码开源
- [ ] 预训练模型
- [x] 训练细节完整（算法伪代码完整，关键参数给出）
- [ ] 数据集可获取（线上数据，无法开放）

---

## 关联笔记

### 基于

- [[DynamoDB]]: 多租户 KV 数据库对比基线，引入 Request Unit 概念
- [[LavaStore]]: ABase DataNode 底层存储引擎（ByteDance 自研，VLDB 2024）
- [[2DFQ]]: 双维公平队列，ABase Dual-Layer WFQ 的理论基础
- [[Prophet]]: 工作负载预测使用的周期性时序模型

### 对比

- [[Cassandra]]: 传统 NoSQL，不支持原生弹性伸缩和细粒度隔离
- [[CosmosDB]]: 微软 Serverless NoSQL，单库 100 万 RU/s 上限
- [[Firestore]]: Google Serverless NoSQL，针对 Web/Mobile 场景

### 方法相关

- [[SA-LRU|Size-Aware LRU]]: DataNode 层缓存策略
- [[AC-LRU|Active-Update LRU]]: Proxy 层缓存策略
- [[Limited Fan-out Hash Routing]]: 代理层热键路由策略
- [[WFQ|Weighted Fair Queueing]]: 多租户公平调度核心机制
- [[请求单元 (RU)|Request Unit]]: 资源量化和计费单位
- [[Resource Pool|资源池]]: ABase 多租户资源管理单元

### 硬件/数据相关

- [[LavaStore]]: ByteDance 自研高性能本地存储引擎

---

## 速查卡片

> [!summary] ABase (SIGMOD-Companion 2025, ByteDance)
> - **核心**: 大规模多租户 NoSQL Serverless 数据库，支撑 13 亿 QPS / 1 EB 存储
> - **方法**: 缓存感知 RU + 两级限流 + Dual-Layer WFQ + AC-LRU/SA-LRU + 集成预测弹性 + 多资源重调度
> - **结果**: 全租户 P99 < SLA 66%，资源利用率 CPU 17%→44%，重调度 RU 标准差 -74.5%，告警 -65%
> - **代码**: 未开源（字节内部系统）

---

*笔记创建时间: 2026-07-20*
