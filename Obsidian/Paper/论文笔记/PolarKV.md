---
title: "PolarKV: Tier Locally, Serve Globally–A KV Cache over Cloud Memory and Storage"
method_name: "PolarKV"
authors: [Yingqiang Zhang, Xinjun Yang, Hao Chen, Feifei Li, Shuwen Wang, Chuan Sun, Sai Wu, Ninglong Weng]
year: 2026
venue: PVLDB 19(12)
tags: [kv-cache, llm-inference, cloud-native, disaggregated-memory, cloud-block-storage, tiered-storage, rdma]
zotero_collection: VLDB/2026
image_source: local
created: 2026-10-10
---

# 论文笔记：PolarKV: Tier Locally, Serve Globally–A KV Cache over Cloud Memory and Storage

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | 阿里云（Alibaba Cloud Computing）、浙江大学 |
| 日期 | 2026，PVLDB 19(12): 4480–4493 |
| 项目主页 | — |
| 对比基线 | [[Mooncake]]、[[3FS]] |
| 链接 | [DOI](https://doi.org/10.14778/3827998.3828047) |
| 原始 PDF | `VLDB/2026/PolarKV.pdf` |

---

## 一句话总结

> PolarKV 打破"整层迁移"的传统分层缓存抽象，把云内存与云块存储按分片两两共置在同一 VM 上做本地化分层，在生产环境将 LLM 推理 TTFT 降低最多 80%。

---

## 核心贡献

1. **云资源带宽–容量–成本耦合的实证分析**：量化云内存与云块存储的带宽/容量/成本耦合特性，给出组合使用云资源构建 KV cache 的实践准则。
2. **shard-paired、co-located 分层架构**：打破"tier-as-a-whole"抽象，把内存层与存储层各自切分为对齐分片，分层迁移变成分片对分片的本地操作，而非跨节点网络搬运。
3. **PolarKV 系统实现**：在每台内存服务节点上直接挂载云块存储卷，复用团队既有的 [[Disaggregated Memory Pool|分布式内存池]] 技术，新增生产级元数据管理与直接块存储集成，避免分布式文件系统开销和专用存储 VM。
4. **大规模云上评估与生产落地**：在微基准、公开长上下文榜单（LooGLE、SCBench）及阿里云真实在线部署（coding-agent 场景）中验证性能与成本收益，已作为独立 KV cache 服务商用。

---

## 问题背景

### 要解决的问题

[[Large Language Model Serving|LLM 推理]]中，[[KV Cache]] 能避免 attention 的重复计算，并通过 [[Prefix Caching|前缀缓存]] 把收益从单请求内扩展到跨请求复用。但 GPU [[High Bandwidth Memory|HBM]] 容量有限，必须把 KV cache 下沉到 host 内存、远端内存或存储。现有系统（Mooncake、AIBrix、LMCache、基于 3FS 的 SSD 方案等）大多在固定本地集群上验证有效，却**没有针对云环境的带宽–容量–成本耦合特性做联合设计**。

### 现有方法的局限

1. **存储不总是便宜**：[[Cloud Block Storage|云块存储]]（CBS，如 AWS EBS、Azure Ultra Disk、阿里云 [[ESSD]]）的带宽与所购容量强耦合（见 Table 1），要拿到多 GB/s 带宽往往要购买 TB 级容量，使高端存储档位的"带宽成本"接近甚至超过内存；而且 CBS 必须挂载在网络带宽足够的 VM 上，带宽越大越要配高规格 VM，进一步推高成本。
2. **云内存可能严重欠利用**：disaggregated memory 的有效带宽受限于宿主 VM 的网络接口（随规格而非内存需求伸缩），即使是百 GB~TB 级 DRAM 配置也可能受限于带宽而非容量；大内存实例往往每 GB 成本更高。
3. **经典分层设计在云上失效**：现有分层方案（见 Figure 3(a)/(b)）分为"并行协调架构"（controller 调度 memory/storage 两个 peer 层）与"严格分层架构"（storage 作为 memory 的 overflow 层），两者都把每层视为全局池，跨层迁移需频繁跨节点网络搬运多 GB 数据；而云网络带宽与 VM 规格绑定、价格昂贵，这一假设在云上直接失效，导致性能下降与过度配置。

### 本文的动机

论文从资源本质上重新归类：**内存是带宽资源，存储是容量资源**。既然 KV cache 工作负载同时需要小热集的高带宽与大冷集的大容量，单一资源无法兼顾，必须要分层；但要让分层真正划算，关键是**避免跨节点网络成为分层迁移的瓶颈**——这正是 PolarKV 的 co-located shard-pair 设计动机。

---

## 方法详解

### 整体架构

Figure 4 展示 PolarKV 的总体结构，由三部分构成：

- **数据层（Data layer）**：内存层（分布式 disaggregated memory）+ 存储层（云块存储），按 shard 两两共置形成分层缓存基质。
- **KV manager server**：管理资源元数据、协调内存/存储分配、向推理引擎暴露统一接口。
- **Proxy**：部署在每个推理节点旁，调度用户请求、负载均衡、利用局部性。

运行时，推理引擎向 manager 查询目标 KV 条目所在的远端内存地址，然后直接通过 [[RDMA]] 对 disaggregated memory 节点读写。

### 核心设计：打破"tier-as-a-whole"抽象

PolarKV 将内存层与存储层**各自切分为对齐的分片（shard）**，每个内存服务 VM 承载一个内存分片，并直接挂载自己的云块存储卷，形成"分片对（shard pair）"。这样，内存–存储间的迁移（fill / eviction / writeback）发生在**同一台 VM 内部**，而不是跨 VM 走网络（Figure 3(c)）。该设计把本来昂贵的"跨节点分层问题"转化为高效的"节点内数据搬运"，消除了额外的 VM 间网络流量，使分层能随云资源弹性平滑扩展。

PolarKV 的内存层基于团队既有的 [[Disaggregated Memory Pool|分布式内存池（DMP）]]（源自 [[PolarDB-MP]] 等前作）：

- **Home Node（HN）**：资源管理与元数据协调。
- **Slab Node（SN）**：贡献实际物理内存资源，推理引擎通过 RDMA 直连 SN。
- **Disk Server（DS）**：与 SN 同机运行，异步地将冷 KV 条目下沉到 [[ESSD]]，按需重新加载。

### 模块 1：Client-side library

客户端库是推理引擎与分布式 KV cache 池之间的桥梁（Figure 5），暴露 `store_kv(token_ids)` / `load_kv(token_ids)` 两个核心接口，内部由两部分组成：

1. **Chunk-based hashing**：把输入 token 序列切成多个 chunk，并采用 [[Chunk-based Prefix-Hash Chain|链式前缀哈希]]——每个 chunk 的 hash 由自身 token id 与前一个 chunk 的 hash 拼接计算而得。这保证两个 chunk 共享同一个前缀 key 的充要条件是其所有前驱 token 完全一致，从而保证前缀匹配的语义正确性。
2. **RDMA client**：通过 RDMA-based RPC 协议向 KV manager 申请远端内存区域并获取物理地址，随后使用 one-sided RDMA 读写直接访问远端内存，低延迟、少 CPU 参与。

### 模块 2：KV cache manager

Figure 6 展示 KV manager 的内部结构，分为三个子模块：

#### Memory manager

初始化时从 disaggregated memory pool 申请一批固定大小的内存 chunk，记录 node ID、起始地址、大小到哈希表。客户端请求内存时，manager 扫描可用 chunk，找到足够空间后切分并更新元数据；每个分配的 KV 条目同时注册进元数据哈希表与 [[LRU]] 链表用于淘汰管理。由于客户端侧 chunk 大小固定，每个 KV 条目大小也固定，简化了分配、减少碎片。节点间采用 round-robin 分配做负载均衡。

#### Metadata management

元数据管理器维护所有 KV 条目的全局映射（key → 内存地址），用一个标志位区分条目在内存还是存储中（on-storage 条目地址置零）。维护两个 LRU 链表（in-memory / on-storage），支持快速命中更新。读请求先向 KV manager 发 RPC 查询地址，命中则客户端直接发起 one-sided RDMA read。后台线程周期性地向存储发出 offload 请求完成内存→存储迁移，并在存储用量超阈值时淘汰 on-storage 的 LRU 条目。为容错，元数据可异步持久化到云数据库，支持 KV manager 崩溃后快速重建元数据表（不需要重建底层 KV 数据本身）。
（？为什么还需要 on-storage，对节点上的数据进行 GC？

#### Write/Read workflow（Figure 7）

写入流程：
1. 客户端向 KV manager 发 RPC 申请远端内存（memory manager 处理）；
2. key 及其内存信息被写入元数据表，并对该 key 加写锁，防止数据未就位前被并发访问或后台淘汰；
3. KV manager 返回分配地址；
4. 客户端执行 one-sided RDMA write 把数据写入远端内存；
5. 客户端发 RPC 释放写锁，完成操作。

读取流程：
1. 客户端向 KV manager 请求地址；
2. manager 查元数据表，若命中则临时加读锁，防止并发写或后台淘汰，返回节点 ID 与地址（否则返回非法地址，表示 cache miss）；
3. 客户端执行 one-sided RDMA read 取数据；
4. 客户端发 RPC 解锁。
（？如果不在内存中呢

#### 并发控制

PolarKV 用**按 key 的锁**处理并发：写操作仅在 key 不存在时分配内存并加写锁，条目只有在客户端完成 one-sided RDMA write 并发 write-finish RPC 后才可读；读操作只在条目存在且未被写锁定时返回地址。这样读者只能观察到完全写完的条目或 cache miss，不会读到部分写入的数据。若多个客户端并发写同一个 key，只有第一个写者继续执行，后续写者收到"重复/进行中"状态并跳过写入——这是安全的，因为**同一模型、同一前缀对应的 KV 条目是确定性**的，从而避免重复分配与重复 RDMA 流量。

#### 可扩展性

KV manager 只参与元数据操作（地址查找、内存分配、锁管理、元数据状态转移），**不在大数据传输路径上**：地址解析后，KV 张量直接在推理引擎与 slab node 之间通过 one-sided RDMA 传输。key-address 表基于哈希，LRU 链表可按 key range 分片以降低高并发下的争用；由于 KV 条目天然按前缀/哈希 key 标识，manager 本身也可按不相交的 key range 分片到多个 manager 实例，支撑更大规模部署。
（看起来是：1. 不同 key range 可以 shard 给不同 manager，manager 管理一组机器，分给哪台机器由 manager 决定）

### 模块 3：Storage layer

PolarKV 的存储层不采用跨多存储节点的分布式文件系统构建全局命名空间，而是采用**shared-nothing、shard-paired**设计（Figure 8）：把存储容量切分为按 SN 对齐的存储分片，每个分片与对应 SN 共置，分层操作局限在该 SN 内。

- **Disk server（DS）**：与所属 SN 同机部署，共享 SN 的内存空间，offload 时可直接访问 in-memory KV 数据，无需额外网络跳转或序列化开销。
- **存储介质选择**：鉴于 KV cache 对带宽和容量都敏感，PolarKV 偏好低每 GB 成本、低每 GB/s 成本的存储档位，再通过聚合多设备带宽弥补单设备带宽不足——实现上用多块低成本、低带宽的阿里云 ESSD **PL0** 组成 [[RAID-0]] 阵列，挂载 ext4 原生文件系统，DS 通过标准文件接口访问 ESSD。
- **文件命名**：DS 不维护额外元数据，直接用约 64 字节的 KV key 哈希值作为文件名；但这不是"一个 token 一个文件"——KV cache 以 chunk 粒度存储，每个 chunk 对应数十到数百 MB 的 KV 数据，单个条目的写入可天然跨越 RAID-0 中的多块 ESSD，充分利用底层聚合带宽，因此无需再进一步切分成更小文件。
	- ？hash 碰撞了怎么办
	- ？如何理解以 chunk 为粒度
- **通信**：DS 与 KV manager 之间同样通过 RDMA-based RPC，DS 内置 RPC handler 处理 offload / load / deletion 请求。

#### Workflow

KV manager 周期性扫描 LRU 链表，为最近未使用的 KV 条目向对应 DS 发出 offload 请求（含 key、内存地址、大小）；DS 启动后台线程异步把指定内存区域写入存储，完成后通知 manager 更新元数据为"on-disk"。（回收内存
当读请求命中的条目位于存储而非内存时，manager 先在配对的 SN 上分配新内存区域，再向 DS 发 load 请求把数据读回内存并确认完成。（读重加载
manager 同时监控各节点存储用量，达到阈值即指示 DS 淘汰最久未用的 on-storage 条目。（总存储量控制

### 模块 4：High availability

PolarKV 的设计哲学是：**KV cache 是性能优化而非正确性关键状态**——数据不可用时推理引擎总能重新计算对应的 attention states，因此 HA 以"保前进、容忍降级"为目标而非强一致复制：

1. **客户端侧故障处理**：KV manager 或内存/存储节点暂时不可用时，客户端库透明地向推理引擎报告 cache miss（进行中的 RPC 中断同样视为 miss），推理引擎正常重计算；客户端持续重试连接，服务恢复后自动恢复使用。
2. **元数据持久化与 manager 恢复**：KV manager 把全局**元数据持久化到外部云数据库**，崩溃重启后从持久化存储重建元数据表；由于**底层 KV 数据本身保持完好，恢复不需要重建**缓存状态。
3. **内存节点故障**：独立处理单节点失效——manager 只失效该节点上的 KV 条目，其他节点上的条目继续可用；故障被限制在小范围数据子集，不产生级联影响。

---

## 关键公式

论文第 2.1 节给出了标准 Transformer 自注意力机制的背景公式，用于解释 KV cache 的动机，但没有提出 PolarKV 自身的新数学公式（系统论文，核心贡献是架构设计而非算法推导）。

### 公式 1: [[Self-Attention|注意力计算]]

$$
\operatorname{Attention}(Q,K,V) = \operatorname{Softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V
$$

**含义**：标准 scaled dot-product attention，解释了为何需要缓存 $K$、$V$ 以避免对历史 token 的重复计算。

**符号说明**：
- $Q = XW_Q,\ K = XW_K,\ V = XW_V$：由隐藏状态 $X$ 线性投影得到的 query/key/value 矩阵；
- $d_k$：key 向量的维度，用于缩放点积以稳定梯度；
- 在自回归解码中，每步只需计算新 token 的 $Q_t,K_t,V_t$，并直接复用已缓存的 $\{K_i\}_{i=1}^{t-1}$、$\{V_i\}_{i=1}^{t-1}$，把每 token 计算复杂度从 $O(n^2)$ 降为 $O(n)$。

---

## 关键图表

> PolarKV PDF 的图形主要为矢量对象，含图/表页面已以高分辨率 PNG 渲染至 `assets/PolarKV/`，完整保留论文中 13 个 Figure 和 3 个 Table 的原始图表、坐标与 caption。

### Figure 1：LLM 推理流水线总览（论文 p2）

![[assets/PolarKV/PolarKV_page3.png]]

**说明**：Prompt 经 [[Autoregressive Generation|prefill]] 生成首个输出 token，随后进入逐 token 的 [[Autoregressive Generation|decode]] 迭代；每轮 decode 都要注意历史全部 token，这正是 [[KV Cache]] 要消除的重复计算来源。

### Figure 2：Prefill 与 Decode 阶段的 Attention 计算（论文 p3）

![[assets/PolarKV/PolarKV_page3.png]]

**说明**：Prefill 阶段一次性计算全部输入 token 的 $Q,K,V$；Decode 阶段每步只需计算新 token 的 $Q_t,K_t,V_t$，并与已缓存的历史 $K,V$ 拼接得到 attention 输出，直观展示 KV cache 消除重复计算的机制。

### Table 1：阿里云 ESSD 块存储档位规格（论文 p4）

| Metric | PL3 | PL2 | PL1 | PL0 |
|---|---|---|---|---|
| Provisioned Bandwidth | min(120+0.5C, 4000) | min(120+0.5C, 750) | min(120+0.5C, 350) | min(100+0.25C, 180) |
| Maximum Bandwidth (MB/s) | 4000 | 750 | 350 | 180 |
| Saturation Capacity (GB) | 7760 | 1260 | 460 | 320 |
| Cost (CNY/GB/month) | 4 | 2 | 1 | 0.5 |
| Cost at Max Bandwidth (CNY/month) | 31040 | 2520 | 460 | 160 |
| Cost per GB/s of Bandwidth (CNY/month) | 7946 | 3441 | 1346 | 910 |

**说明**：$C$ 为存储容量（GB）。随着档位升高，带宽上限提升但每 GB/s 带宽的边际成本并不线性下降（PL3 反而是每 GB/s 最贵的档位），说明"高带宽存储"并不总是比内存便宜，为 PolarKV 选用 PL0 做 RAID-0 聚合提供了直接依据。

### Table 2：阿里云 ECS 实例配置与成本（论文 p4）

**说明**：对比内存优化型（r9i 系列）与计算优化型（c8i 系列）实例在不同规格下的 vCPU、内存、网络带宽及月度成本。关键发现是：内存容量与网络带宽不成比例扩展——memory-optimized 实例的"每 GB/s 带宽成本"随规格增大而显著上升（r9i.large 为 1024.13 CNY/月，r9i.48xlarge 升至 4000.21 CNY/月），而 compute-optimized 实例的每 GB/s 带宽成本更低，是 PolarKV 为存储托管 VM 选择 c8i 系列的依据（见 Table 3）。

### Figure 3：主流分层缓存架构 vs. PolarKV 的共置集成设计（论文 p5）

![[assets/PolarKV/PolarKV_page5.png]]

**说明**：(a) 并行协调架构——memory/storage 作为 peer 层，由 coordinator 调度；(b) 严格分层架构——storage 作为 memory 的 overflow 层；(c) PolarKV 的 co-located 设计——内存与存储按分片两两共置在同一 VM 内，分层流量不出 VM。(a)(b) 都把每层视为全局池，跨层迁移需走网络；(c) 把跨节点问题转化为节点内问题。

### Figure 4：PolarKV 总体架构（论文 p5）

![[assets/PolarKV/PolarKV_page5.png]]

**说明**：Proxy 调度用户请求至推理引擎，推理引擎通过 PolarKV client 和 RDMA 访问 KV cache manager 与分布式 disaggregated memory（Home Node + Memory Slab Node）；每个 Slab Node 旁的 Disk Server 把冷 KV 卸载到共置的云块存储（ESSD）。

### Figure 5：Client-side 架构（论文 p6）

![[assets/PolarKV/PolarKV_page6.png]]

**说明**：推理引擎调用 `store_kv` / `load_kv`，经 chunk-based hashing 模块产生 key，再由 RDMA client 向 KV manager 请求地址（`allocate_mem`/`get_addr`），随后直接对 memory slab node 执行 `remote_write`/`remote_read`。

### Figure 6：KV cache manager 架构（论文 p7）

![[assets/PolarKV/PolarKV_page7.png]]

**说明**：RPC handler 接收客户端的 allocate/lookup 请求；内部由 Memory manager（Allocator + 空闲 chunk 表）、Metadata manager（in-memory/on-storage 两个 KV 链表 + 哈希表）、Storage manager 和 Cluster manager 协同，Metadata 作为枢纽连接内存管理与存储管理。

### Figure 7：内存数据访问的 Write/Read Workflow（论文 p7）

![[assets/PolarKV/PolarKV_page7.png]]

**说明**：(a) 写流程：①申请内存 ②写入元数据+加写锁 ③返回地址 ④one-sided RDMA write ⑤释放写锁；(b) 读流程：①请求地址 ②元数据查找+加读锁+返回地址 ③one-sided RDMA read ④解锁。RPC 走 RDMA-based RPC，大数据传输走 one-sided RDMA。

### Figure 8：存储层架构（论文 p8）

![[assets/PolarKV/PolarKV_page8.png]]

**说明**：Memory Slab Node 内的 Disk server 与 KV manager 通过网络交换 read/write/delete 请求和响应；Disk server 内部先经原生文件系统（ext4）再落到由多块 ESSD 组成的 RAID-0 阵列，兼顾聚合带宽与标准文件接口的简洁性。

### Figure 9：微基准下 PolarKV 整体性能（论文 p9）

![[assets/PolarKV/PolarKV_page9.png]]

**说明**：在 10K/20K/30K token 三种 prompt 长度、并发 1–4 下对比"无外部 cache"、"PolarKV-memory"（纯内存，3TB）、"PolarKV"（混合内存+存储，480GB 内存+每节点约1.2TB 存储）。PolarKV 与 PolarKV-memory 的 TTFT 几乎一致，且相对无缓存基线最多降低 TTFT 80%，说明在该环境下混合设计已贴近纯内存的性能上限（因为 KV 加载主要受网络带宽限制而非 DRAM/存储介质峰值带宽限制）。

### Figure 10：KV cache 带宽对 TTFT 的影响（论文 p10）

![[assets/PolarKV/PolarKV_page10.png]]

**说明**：20K-token 输入下，随 PolarKV 可用带宽从 3 GB/s 增至 16 GB/s，TTFT 先急剧下降后趋于平缓；约 12 GB/s 后收益边际化，说明超过某阈值后瓶颈转移到 GPU 侧网络限制等其他环节，强调带宽（而非存储容量）才是 KV cache 效率的首要决定因素。

### Table 3：固定云资源预算下 3FS、Mooncake、PolarKV 的配置（论文 p10）

| 系统 | Memory Capacity | Storage Capacity | Bandwidth | Total cost (per month) |
|---|---|---|---|---|
| 3FS | — | 21.4 TB | 12.09 GB/s | 18,352.30 CNY |
| Mooncake | 928 GB | — | 17.98 GB/s | 18,413.84 CNY |
| PolarKV | 156 GB | 21.4 TB | 12.09 GB/s | 18,352.30 CNY |

**说明**：在同等月度预算下，3FS 用 67 块 320GB PL0 ESSD 条带化达到约 12 GB/s（约 21TB 容量，需 39 台 VM 承载）；Mooncake 用低每 GB 成本的 r9i.large 实例堆出 58 台 VM，得到 928GB 内存与 17.98GB/s 带宽；PolarKV 用低每 GB/s 带宽成本的 c8i.large 承载同样的 21.4TB 存储加 156GB 内存，带宽与 3FS 相同但多了内存层。

### Figure 11：LooGLE 与 SCBench 上的性能（论文 p10）

![[assets/PolarKV/PolarKV_page10.png]]

**说明**：(a) LooGLE 的 shortdep_qa / shortdep_cloze / longdep_qa 三个子集；(b) SCBench 的 Code.RepoQA / En.MultiChoice / En.Sum / Mix.Sum+NIAH / Mix.RepoQA+KV 子集。各图对比"无外部 cache"、3FS、Mooncake、PolarKV 四种配置的 TTFT；PolarKV 在几乎所有子集上显著领先，尤其在大 KV footprint 的 shortdep_cloze / longdep_qa 上，相对 3FS 和 Mooncake 降低 TTFT 41%–58%。

### Figure 12：生产环境 Canary 灰度上线结果（论文 p11）

![[assets/PolarKV/PolarKV_page11.png]]

**说明**：某自动驾驶领域客户的 coding-agent 服务（SGLang + GLM-4.7），50% 流量分流至 40×H20 GPU + 6TB PolarKV 的新集群，另 50% 留在原 216×H20 GPU 无外部缓存集群。结果：P90 延迟从 49.73s 降至 24.91s，P50 从 13.88s 降至 6.81s，平均延迟降低 42%——用不到五分之一的 GPU 数量取得明显更低延迟，核心驱动力是 KV cache 命中率从约 30%（节点本地）提升到约 85%（跨节点共享）。

### Figure 13：全量迁移前后的性能对比（论文 p11）

![[assets/PolarKV/PolarKV_page11.png]]

**说明**：全量迁移后，集群由 256×H20 GPU 缩减为 160×H20 GPU + 20TB 存储，同时服务流量（token 数）增长约 68%（全天 1.9B→3.2B，高峰 4 小时 0.8B→1.5B）。即便如此，PolarKV 全天平均 TTFT 降低 55%、平均端到端延迟降低 45%；高峰 4 小时平均 TTFT 降低 61%、平均延迟降低 53%；P50/P90 延迟降低 48%–78%，验证了"更少 GPU + 更高吞吐 + 更低延迟"同时达成。

---

## 实验

### 实验环境

| 项目 | 内容 |
|---|---|
| 云平台 | 阿里云公共云，GPU/内存/存储资源均在云上实测 |
| GPU | 每台 GPU 实例配 8×H20 96GB |
| 推理引擎 | vLLM、SGLang |
| 模型 | DeepSeek-R1（微基准）、Qwen3-235B-A22B（LooGLE/SCBench）、GLM-4.7（生产案例） |

### 基线

- [[Mooncake]]：代表 memory-centric KV cache（跨主机池化 DRAM）。
- 基于 [[3FS]] 的 DFS KV cache：代表 storage-centric KV cache（通用分布式文件系统，被 SGLang 等推理框架支持作为 KV cache backend）。

论文说明：目前没有已知的开源外部 KV cache 系统采用与 PolarKV 相同的混合内存–存储设计点，因此这两个基线代表存储中心、内存中心两种实践路线，与 PolarKV 的混合路线三方对照。

### 关键实验结论

1. **微基准（5.2 节）**：PolarKV 与纯内存版 PolarKV-memory 的 TTFT 几乎一致，相对无外部缓存基线降低 TTFT 最多 80%；原因是 KV 加载主要受网络带宽（GPU host 双 100-Gbps NIC，合计 200-Gbps）限制，而非 DRAM/存储介质峰值带宽限制，经过充分带宽配置的混合层级可以达到接近纯内存的加载延迟。
2. **带宽敏感性（5.3 节）**：TTFT 随存储带宽从 3GB/s 提升到 12GB/s 急剧下降，12GB/s 后收益边际化——带宽配置是预测性低延迟的关键，而非一味追求更大容量。
3. **公开长上下文基准（5.4 节）**：固定云资源预算下，PolarKV 在 LooGLE 上相对 3FS 降低 TTFT 47%–53%，相对 Mooncake（在大 footprint 场景下）降低 41%–58%；在 SCBench 上相对 3FS 降低 47%–58%、相对 Mooncake 降低 3%–58%。KV footprint 较小时 Mooncake 接近 PolarKV，footprint 增大后 Mooncake 的容量瓶颈凸显（在 shortdep_cloze 甚至不如 3FS）。
4. **生产案例（5.5 节）**：canary 阶段用不到 1/5 的 GPU 取得近 2× 的 P50/P90 延迟改善；全量迁移后用 37.5% 更少的 GPU 服务 68% 更高的流量，仍将全天平均 TTFT/延迟降低 55%/45%，峰值期降低 61%/53%，P50/P90 降低 48%–78%。核心机制是跨节点 KV 共享把命中率从约 30%（节点本地 host memory）提升到约 85%（PolarKV 共享池）。

---

## 批判性思考

### 优点

1. **问题刻画扎实、源于一线成本数据**：用阿里云 ESSD/ECS 真实计价表（Table 1、2）定量证明"存储未必便宜""内存可能欠利用"，而非泛泛而谈云资源弹性。
2. **架构洞察清晰且可泛化**："tier-as-a-whole → shard-paired co-located" 的转变直指云环境分层缓存失效的根因（跨节点网络带宽昂贵且与 VM 规格绑定），该思路对任何需要跨异构云资源分层的系统（不止 KV cache）都有参考价值。
3. **工程取舍务实**：HA 设计明确把 KV cache 定位为"可丢失的性能优化层"，避免过度工程化的强一致复制，契合 LLM 推理场景的实际语义（随时可重算）。
4. **真实大规模生产验证**：罕见地给出了完整的 canary→全量迁移前后对比和 token 吞吐数据，具有很强的工业参考价值，是目前公开的、关于生产级 coding-agent 场景 KV cache 行为的首批案例之一。

### 局限性

1. **缺乏同类系统的直接对比**：论文自己承认没有开源的"同样混合设计点"的外部 KV cache 系统可比，所以只能分别和纯内存（Mooncake）、纯存储（3FS）比较，PolarKV 相对"其他设计优良的混合方案"的优势未知。
2. **元数据服务的单点/扩展性论证偏理论**：论文描述了 KV manager 可按 key range 分片以应对更大规模部署，但没有给出分片后跨 manager 的协调、负载均衡或故障恢复的实验数据。
3. **存储带宽的 RAID-0 聚合依赖多设备叠加，故障域扩大**：用多块低成本 ESSD 做 RAID-0 提升聚合带宽的同时，意味着任一块盘故障即可能导致整条带的数据不可用；论文的 HA 讨论聚焦节点级失效，未细化单块 ESSD 故障对 RAID-0 条带的影响及恢复路径。
4. **"co-located"是否要求内存与存储 1:1 固定配比**：论文的分片两两共置设计隐含每个内存节点必须搭配固定比例的本地存储，这对不同 workload 的热/冷比例差异较大的场景（如突发极热点或极冷归档）弹性调整能力有限；论文未讨论跨 VM 的"分片再平衡"机制。
5. **评估的带宽阈值（12 GB/s）与具体云环境强相关**：该阈值依赖于论文测试环境的具体 GPU host 网络规格（200-Gbps 聚合），在不同网络拓扑/实例规格下，"够用带宽"的具体数值可能完全不同，复现时需按自身环境重新测算。

### 潜在改进方向

1. 补充与其他云原生混合 KV cache（如内部对比设计点变体）或学术原型的横向评测，弱化"自己定义基线组合"的局限。
2. 探索内存/存储分片比例的自适应配置（根据观测到的热/冷比动态调节），而非固定的 shard-pair 配比。
3. 针对 RAID-0 存储层设计更细粒度的故障隔离或纠删码方案，弱化单盘失效对整条带的影响。
4. 结合 [[KV Cache Recomputation|重计算成本模型]]，在 HA 决策中显式权衡"重算成本 vs. 跨节点迁回成本"，而非一刀切地把失效都退化为 cache miss。

### 可复现性评估

- [ ] 代码开源
- [ ] 预训练模型（不适用，系统论文）
- [x] 训练/部署细节（实例类型、带宽、成本配置）较为完整
- [x] 公开数据集（LooGLE、SCBench）可获取；生产案例数据不可获取

---

## 关联笔记

### 基于

- [[Disaggregated Memory Pool|分布式内存池（DMP）]]：PolarKV 内存层直接复用团队前作的 disaggregated memory 技术。
- [[PolarDB-MP]]：PolarKV 内存层所基于的多主云原生数据库共享内存前作之一。
- [[RDMA]]：PolarKV 元数据 RPC 与大数据传输的底层通信机制。
- [[KV Cache]]、[[Prefix Caching]]：PolarKV 面向的核心缓存抽象。

### 对比

- [[Mooncake]]：memory-centric 远端内存 KV cache 基线，容量受限但带宽充足时表现接近 PolarKV。
- [[3FS]]：storage-centric、基于分布式文件系统的 KV cache 基线，访问延迟和元数据开销更高。

### 方法相关

- [[Shared-Storage Architecture]]：PolarKV 的共享/共置资源组织思想与之呼应，但方向相反（存储与计算共置于同一 VM 而非完全解耦）。
- [[Hot-Cold Tiering]]：PolarKV 的内存–存储分层本质上是 LLM 场景下的热/冷数据分层。
- [[ESSD]]：PolarKV 存储层所选用的阿里云块存储产品，论文据其带宽–容量耦合特性制定选型准则。
- [[RAID-0]]：PolarKV 用多块低带宽 ESSD 聚合出高带宽存储层的实现手段。
- [[Chunk-based Prefix-Hash Chain|链式前缀哈希]]：PolarKV 客户端用于前缀感知 KV 索引的核心数据结构机制。
- [[High Bandwidth Memory|GPU HBM]]：论文开篇指出的容量瓶颈来源，是 KV cache 下沉到 host/远端内存/存储的直接动机。
- [[PagedAttention]]、[[Self-Attention]]：LLM 推理与 KV cache 的基础技术背景。

### 硬件/数据相关

- [[LooGLE]]、[[SCBench]]：论文使用的公开长上下文/异构推理工作负载基准。
- [[Internet of Vehicles]]（跨论文对照）：另一类对云资源耦合特性敏感的大规模工作负载场景（Lindorm-UWC），与 PolarKV 面对的"带宽-容量-成本耦合"问题在方法论上有相通之处。

---

## 速查卡片

> [!summary] PolarKV
> - **核心**: 打破"整层迁移"抽象，把云内存/存储按分片两两共置在同一 VM 内做本地化分层，避免跨节点网络成为分层瓶颈。
> - **方法**: shard-paired co-located 架构 + RDMA 分布式内存池 + ESSD RAID-0 存储层 + per-key 锁与异步元数据持久化 HA 设计。
> - **结果**: 微基准最多降 TTFT 80%；LooGLE/SCBench 相对 3FS/Mooncake 降 TTFT 3%–58%；生产全量迁移用 37.5% 更少 GPU 服务 68% 更高流量，平均延迟降 45%–53%。
> - **代码**: 未开源，已作为阿里云商用独立 KV cache 服务。

---

*笔记创建时间: 2026-10-10*
