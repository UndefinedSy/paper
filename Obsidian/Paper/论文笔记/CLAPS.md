---
title: "CLAPS: A Load-Aware Proxy Resource Pooling System for Reducing Resource Redundancy in Large-Scale Cloud Storage"
method_name: "CLAPS"
authors: [Xiuqi Huang, Shiwei Cao, Honghao Wang, Ruijie He, Huameng Zhou, Cheng Wang, Linyu Chen, Qingsong Zhu, Shunxin Min, Yunlong Wang, Xunzhi Xu, Yangming Chen, Chen Ling, Bingwen Yuan, Yushan Sun, Maoyin Liu, Chengyu Dong, Rui Shi, Wei Chen]
year: 2026
venue: SIGMOD
tags: [cloud-storage, proxy-pooling, load-balancing, distributed-storage, resource-management]
zotero_collection: SIGMOD/2026
image_source: local
created: 2026-07-14
---

# 论文笔记：CLAPS: A Load-Aware Proxy Resource Pooling System for Reducing Resource Redundancy in Large-Scale Cloud Storage

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | Zhejiang University, ByteDance Inc. |
| 日期 | May 2026 |
| 项目主页 | N/A |
| 对比基线 | [[Fisc]]、传统静态 Proxy-Storage 绑定 |
| 链接 | [ACM DL](https://doi.org/10.1145/3788853.3803082) |

---

## 一句话总结

> CLAPS 将孤立的有状态 ProxyServer 集群整合为共享弹性资源池，通过两层负载感知写调度和直接路径读优化，消除大规模云存储中静态 Proxy-Storage 绑定导致的资源冗余。

---

## 核心贡献

1. **首个生产级有状态 Proxy 池化系统**: 设计并实现 CLAPS，在保持高性能和可用性的同时，消除静态 Proxy-Storage 绑定导致的资源冗余
2. **两层负载感知写调度**: 耦合动态写 Blob 分配、机架感知 [[ProxyServer]] 分配和基于冻结的 Group 再平衡，最小化迁移开销并保证写稳定性
3. **直接路径读优化三件套**: One-Hop 读机制消除代理转发、[[TierTable]] 辅助读加速近期写入数据访问、黑名单容错保障高可用

---

## 问题背景

### 要解决的问题

在 ByteDance 的生产环境中，[[ByteStore]] 与 [[ByteTier]] 构成分布式存储底座，其上层服务（TOS、TLS、RMQ）通过 ProxyServer 集群统一访问。然而，**每个 ProxyServer 集群静态绑定到一个特定的 ByteStore 集群**，必须按该集群的峰值负载进行资源预配。

生产观测数据表明：不同集群的负载存在显著的时空异质性（如图 Fig. 1，集群 A/B/C/D 的平均吞吐分别为 13.07/7.71/3.82/15.50 GB/s），各集群的峰谷时间不同步，**整体需求远低于各集群峰值之和**，造成大量 Proxy 资源冗余浪费。

### 现有方法的局限

- **静态绑定**: 无法实现跨集群弹性资源共享，大量 Proxy 资源在非峰值时段空闲
- **[[Fisc]]**: 利用共享内存改进转发路径，但仍需代理参与读路径，存在两跳转发开销
- **[[DiSC]]**: 采用有状态内核连接绕过代理转发，但依赖全局同步序列，实现复杂且灵活性受限

### 本文的动机

[[ByteTier]] 的两阶段写机制要求代理层维护跨请求的瞬态写状态（[[TierTable]] 数据、[[WAL]] 文件关联），使得 Proxy 资源无法像无状态代理那样自由重分配。CLAPS 要在保持有状态语义正确性的前提下，实现 Proxy 资源的弹性池化。

---

## 系统架构

### 整体架构（Figure 2）

![[CLAPS_fig2_architecture.png]]

CLAPS 是上层存储服务与底层存储基础设施之间的统一代理池化中间件，由两大组件构成：

**LightSDK Client**（用户侧 I/O 入口）
- 暴露简化存储 API，不含路由逻辑
- **Client Cache**: 缓存近期访问的 [[ChunkServer]] 地址，支持 One-Hop 直接读
- **Blacklist**: 临时排除故障 ChunkServer，避免重复访问错误节点

**Proxy Pool**（操作核心）由三个互补子组件构成：

- **Global Proxy Coordinator**: 系统级决策中心，维护跨集群负载、资源可用性和故障域的全局视图，指示各 ProxyMaster 进行弹性扩缩
- **ProxyMaster**: 每个 ByteStore 集群的管理者，接收 Global Coordinator 指令，执行路由表更新、ProxyServer 状态监控和 Group 迁移
- **ProxyServer**: 分为 RW（读写）和 RO（只读）两种类型：
  - **RW ProxyServer**: 处理读写请求，持有 TierTable 缓存近期写入数据
  - **RO ProxyServer**: 专为读请求设计，轻量高效，从 MetaServer 获取元数据服务读

最细粒度的服务单元是 **group**，每个 Blob 写请求在特定 group 中处理，支持 group 粒度的冻结和跨 ProxyServer 迁移。

### 端到端读写流程

**写流程（Write Workflow）**：
1. LightSDK Client 发送请求至 ProxyServer，触发 ProxyMaster 资源调度
2. ProxyMaster 根据 Monitor 报告的负载选择低负载 RW ProxyServer，分配唯一 write group
3. Group 与 ByteStore 性能池中的 [[WAL]] 文件绑定，写命令下发给选定的 RW ProxyServer
4. Client 将写数据发送给 RW ProxyServer：数据缓存于 [[TierTable]]，WAL 追加到绑定文件
5. WAL 追加成功后数据视为持久化；异步地，TierTable 满足刷盘条件后将数据 flush 到 ByteStore 容量池

**读流程（Read Workflow）**：
1. ProxyMaster 维护路由表并推送至 LightSDK Client
2. Client 优先使用 RO ProxyServer 查询 MetaServer 获取 Blob 的物理位置元数据
3. Client 检查 Client Cache：命中则直接 One-Hop 访问 ChunkServer；未命中则走 ProxyServer 转发
4. RO ProxyServer 将 ChunkServer 地址同步回 Client 后直接重定向；RW ProxyServer 优先从 TierTable 返回数据

---

## 方法详解

### 模块一：两层负载感知写调度（Two-Tier Write Scheduling）

如 Fig. 3 所示，CLAPS 在 ProxyServer 间实现跨层次写调度，Groups 存在两种状态：**open**（接受新写 Blob 分配）和 **frozen**（不调度新 Blob，但继续处理已分配 Blob 的写请求）。

#### 4.1 动态写 Blob 分配（Dynamic Write Blob Assignment）

**背景观察**：约 80% 的写负载来自高吞吐写 Blob，静态哈希分配会将这些高流量 Blob 集中到少数 ProxyServer，引发负载倾斜并触发代价高昂的 Group 迁移。

**设计**：
- 低负载时：按哈希映射将写 Blob 分配给 Group
- 高负载时：ProxyMaster 持续聚合负载统计，**动态排除 Top 15% 过载 ProxyServer** 出候选集
- 写分配流程分为无状态的 Touch Phase（分配 blob-group 映射）和有状态的 Write Phase，两阶段可在不同 ProxyServer 上独立处理，降低节点争用

#### 4.2 机架感知 ProxyServer 分配（Rack-Oriented ProxyServer Allocation）

**代价矩阵模型**：用矩阵 $M \in \mathbb{Z}^{n \times m}$ 表示分配状态，$n$ 为 ByteStore 集群数加未分配行，$m$ 为物理机架数，$M_{ij}$ 为分配给集群 $i$ 位于机架 $j$ 的 ProxyServer 数量。

**代价函数（Cost Function）**：

$$
\text{Cost}(M) = \sum_{i=1}^{n} \varphi_i \sum_{j=1}^{m} \left(\frac{M_{ij}}{\sum_{j=1}^{m} M_{ij}}\right)^2 + \psi \cdot \text{hash}(i, j) \cdot M_{ij}
$$

约束条件：$\psi \cdot \text{hash}(i, j) \ll \dfrac{\varphi_i}{\left(\sum_{j=1}^{m} M_{ij}\right)^2}$

**含义**: 最小化代价函数实现两目标的平衡：分散（Dispersion）减少单机架故障影响，亲和性（Affinity）减少 ProxyServer 重分配时的迁移开销。

**符号说明**：
- $\varphi_i$：集群 $i$ 的优先级权重（未分配资源取更小值）
- $\psi \cdot \text{hash}(i, j)$：机架亲和性权重，保证重分配时部分 ProxyServer 保持不变
- 分散项：$\sum_j (M_{ij}/\sum_j M_{ij})^2$ 在均匀分布时达到理论最小值

**约束条件**：
- 机架容量：$\sum_i M_{ij} = \text{RackResources}(j)$，每机架 ProxyServer 总数固定
- 集群配额：$\sum_j M_{ij} = \text{ClusterQuota}(i)$，每集群分配上限
- 分散界限：每集群 $i$，$|\{j | M_{ij} > k\}| \le \max(\sum_j M_{ij} - k \cdot \text{MinRacks}(i), 0)$，限制单机架集中度

**[[MCMF]] 求解**：分配问题转化为[[最小费用最大流]]（Minimum-Cost Maximum-Flow）问题。构建流网络（Source → cluster-front → cluster-back → rack → free-proxies → Sink），使用 [[SPFA]]（Shortest Path Faster Algorithm）迭代找最短增广路。

**动态模式切换**（Algorithm 1）：
- 正常负载：**快速分配模式**，仅考虑空闲 ProxyServer
- 空闲资源低于阈值 $\tau$：**后台回收模式**，触发资源回收
- 资源紧缺：**联合模式**，综合评估分配/回收/补充三种动作，选最低代价方案

#### 4.3 基于冻结的 Group 再平衡（Freeze-Based Group Rebalancing）

**ProxyServer 重分配两阶段**：
1. **回收（Reclaim）**：Global Coordinator 向目标集群 ProxyMaster 发送 Reclaim 指令，后台迁移写流量完成后释放 ProxyServer
2. **重分配（Claim）**：Global Coordinator 向目标集群 ProxyMaster 发送 Claim 指令，ProxyServer 仅在空闲或已被该集群占用时返回成功

**Group 迁移策略**：
- **优先迁移 Frozen Groups**：冻结 Group 不再接受新 Blob 分配，残余流量递减，迁移代价远低于 Active Group
- 生产部署中维持约 **50%** Group 处于冻结状态，在迁移灵活性与前台吞吐间取得平衡
- **优先迁移"最冷" Group**：即冻结时长最长的 Group，最大程度减少对在线服务的干扰
- 扩容时，负载最高的 Top 15% ProxyServer 上的 Group 拒绝接受新写 Blob，防止残余流量加剧负载不均

---

### 模块二：直接路径读访问优化（Direct-Path Read Access Optimizations）

#### 5.1 One-Hop 读机制（One-Hop Read Mechanism）

如 Fig. 4 下部所示，Read Workflow 优化：

1. CLAPS 启动时，Client 向 ProxyMaster 获取路由表缓存本地
2. 首次读：Client 哈希路由选 ProxyServer（优先 RO），ProxyServer 查询 MetaServer 获得目标 Blob 的 [[ChunkServer]] 地址（含 Chunk 索引、偏移、长度、磁盘 ID、网络地址）
3. Client 将 ChunkServer 地址存入 **Client Cache**
4. 后续读：直接命中缓存，**绕过 ProxyServer**，直接从 ChunkServer 的 RPC API 拉取数据
5. 直接路径失败时，Client 假设 Chunk 迁移，重新从 ProxyServer 获取更新的位置信息；再失败则触发 [[EC 重构读]]或切换副本

#### 5.2 TierTable 辅助读加速（TierTable-Assisted Read Acceleration）

如 Fig. 4 上部所示，针对**频繁写入导致数据尚未回写 ByteStore** 的场景：

- Client 直接识别持有目标 Blob 数据的 RW ProxyServer，从其 **TierTable** 读取数据
- 绕过等待 ByteStore 回写的延迟，避免内部重定向开销
- **读取顺序**：对最后一个 Chunk Group，从高地址到低地址串行读取（先读 [4M, 8M) 再读 [1M, 4M)），与 TierTable 活跃数据位于高地址的机制对齐
- **TierTable 生命周期策略**：主动将数据尽快下刷到存储层，不为读而延长 TierTable 存活时间，长期降低对 Proxy 辅助读的依赖

**一致性保障**：读请求处理前执行 Open 过程，从 MetaServer 获取文件元数据：
- 无 WAL 关联时：consolidated-length 为最终值，此范围内的读可绕过写节点
- 有 WAL 关联时：MetaServer 返回长度 0，走替代读路径

#### 5.3 黑名单容错机制（Blacklist-Based Fault Tolerance）

**ProxyServer 故障**：I/O 超时时重试 3 次，确认失败后通知 ProxyMaster 将 Client 重定向到备用 ProxyServer。

**ChunkServer 故障**（直接路径读场景的核心问题）：

- **双层黑名单**：
  - **节点级黑名单**：针对通信相关故障（RPC 错误等）
  - **磁盘级黑名单**：针对孤立磁盘错误
  - 分离设计避免将整个节点因单个故障磁盘而屏蔽，防止吞吐量下降

- **黑名单触发规则**：
  - RPC 超时 → 仅对应磁盘加入磁盘级黑名单
  - 其他 RPC 错误 → 整个节点加入节点级黑名单

- **过期策略**：基于时间的过期算法，过期时间较长，减少黑名单频繁波动

---

## 关键公式

### 公式1：[[代价函数|Proxy 分配代价函数]]

$$
\text{Cost}(M) = \sum_{i=1}^{n} \varphi_i \sum_{j=1}^{m} \left(\frac{M_{ij}}{\sum_{j=1}^{m} M_{ij}}\right)^2 + \psi \cdot \text{hash}(i, j) \cdot M_{ij}
$$

约束：$\psi \cdot \text{hash}(i, j) \ll \dfrac{\varphi_i}{\left(\sum_{j=1}^{m} M_{ij}\right)^2}$

**含义**：量化 ProxyServer 分配质量，平衡机架分散（降低故障影响范围）和机架亲和性（减少迁移开销）。

**符号说明**：
- $M \in \mathbb{Z}^{n \times m}$：分配状态矩阵，$M_{ij}$ 为集群 $i$ 在机架 $j$ 上的 ProxyServer 数
- $\varphi_i$：集群 $i$ 的优先级权重
- $\psi \cdot \text{hash}(i, j)$：机架亲和性权重，防止重分配时大规模迁移
- 分散项：取值越小表示分布越均匀，均匀分布时达最小值

---

## 关键图表

### Figure 1：生产集群工作负载变化

**说明**：展示 4 个 ByteStore 集群（A: 13.07 GB/s、B: 7.71 GB/s、C: 3.82 GB/s、D: 15.50 GB/s）在 72 小时内的流量变化。各集群峰谷不同步，整体实际需求远低于各集群峰值之和，是 Proxy 资源冗余的根本动因。

### Figure 2：CLAPS 整体架构

**说明**：CLAPS 的完整系统架构，包含 LightSDK Client（含 Client Cache 和 Blacklist）和 Proxy Pool（Global Proxy Coordinator、ProxyMaster、RW/RO ProxyServers）。底层 ByteTier 提供两阶段写，ByteStore 提供 ChunkServer 和 MetaServer 接口。

### Figure 3：两层负载感知写调度

**说明**：两层调度的示意图。微观层面：写 Blob 动态分配到 open Groups；宏观层面：ProxyServer 跨 ByteStore 集群重分配；完成后 ProxyMaster 执行 Group 再平衡（冻结 Group 迁移）。

### Figure 4：One-Hop 和 TierTable 辅助读工作流

**说明**：
- 上半部（TierTable 辅助读）：Client 直接读 RW ProxyServer 的 TierTable，适用于频繁写入的热数据
- 下半部（One-Hop 读）：Client 首次读经 ProxyServer 获取 ChunkServer 地址，后续通过 Client Cache 直接访问 ChunkServer

### Figure 5：动态写 Blob 分配性能对比

| 指标 | 传统哈希 | CLAPS Dynamic | 改善幅度 |
|------|---------|---------------|---------|
| ProxyServer 负载标准差（最大值）| ~60M | <50M | **24.61%** |
| 热点负载比例（>125%）| 11.7% | 4.5% | ↓7.2pp |
| 理想负载比例（75%-125%）| 76.9% | 89% | ↑12.1pp |
| P99 写延迟 | 24.01 ms | 22.29 ms | **7.16%** |

### Table 1：多集群流量变化场景

| 阶段 | Cluster 1 | Cluster 2 |
|------|-----------|-----------|
| Initial | 3 Gbps | 3 Gbps |
| Surge 1 | 15 Gbps | 3 Gbps |
| Recovery 1 | 3 Gbps | 3 Gbps |
| Surge 2 | 3 Gbps | 15 Gbps |
| Recovery 2 | 3 Gbps | 3 Gbps |

### Figure 6：两集群突增-恢复周期中的写性能和 ProxyServer 分配

**说明**：
- (a) 延迟对比：流量突增时写延迟短暂上升，ProxyServer 调度机制快速稳定
- (b) 吞吐对比：集群吞吐迅速扩展，充分利用跨集群 Proxy 资源
- (c) ProxyServer 分配：空闲 ProxyServer 快速重分配到过载集群，验证 Proxy 池化的响应性

### Figure 7：不同工作负载配置下 One-Hop 读 vs 代理读延迟对比

| 工作负载配置 | 代理读延迟 | One-Hop 读延迟 | 绝对减少量 |
|-------------|-----------|--------------|-----------|
| 20G × 100 文件 | ~1250 μs | ~900 μs | **904.47 μs** |
| 10G × 200 文件 | ~1750 μs | ~1100 μs | **703.37 μs** |
| 5G × 400 文件 | ~2400 μs | ~1400 μs | **1030.89 μs** |
| 2G × 1000 文件 | ~5000 μs | ~3800 μs | **1197.68 μs** |
| 1G × 2000 文件 | — | — | **804.58 μs** |

### Table 2：TierTable 辅助读场景分类

| 模式 | 数据来源 | 适用场景 |
|------|---------|---------|
| TierTable Only | 全部来自缓存 | 活跃写入阶段 |
| Hybrid | 部分缓存 + One-Hop 读 | 混合可用性 |
| Pure One-Hop | 全部 One-Hop 读 | 数据已写入 ByteStore |

### Figure 8：TierTable Only vs Hybrid vs One-Hop Only 读延迟对比

**关键发现**：
- TierTable-only 相比 Pure One-Hop：延迟减少最高达 **1996.21 μs**（1G 配置），20G 配置下减少 **743.32 μs**
- Hybrid 相比 Pure One-Hop：延迟减少最高达 **1190.53 μs**（1G 配置），20G 配置下减少 **483.89 μs**
- 在网络带宽受限的场景下，TierTable 优势尤为明显

### Figure 9：CLAPS 24 小时生产工作负载追踪

| 指标 | 平均值 | 峰值 |
|------|--------|------|
| 写吞吐 | 300.44 MB/s | 734.20 MB/s |
| 写 IOPS | 3197.71 ops/s | 8740.00 ops/s |
| 写延迟（最大）| — | 3.55 ms |
| 读吞吐 | 0.13 MB/s | 4.30 MB/s |
| 读 IOPS | 1192.31 ops/s | 1900.00 ops/s |
| RW Blobs/s | 87.86 blobs/s | 156.00 blobs/s |

**关键发现**：系统整体呈写主导型工作负载；读吞吐持续偏低，说明大量读请求已通过 One-Hop 路径直接访问 ChunkServer，证明直接路径读优化的实际效果。

---

## 实验

### 实验环境

- ByteDance 生产环境，真实生产工作负载
- 多个 ByteStore 集群，6 个业务存储集群（Section 6.3）
- 22-ProxyServer 集群（TierTable 实验，512 KB block size，4 threads）

### 实验设计

| 实验 | 基线 | 关注指标 |
|------|------|---------|
| 动态写 Blob 分配 | 传统哈希分配 | 负载标准差、热点比例、P99 延迟 |
| 多集群流量突增 | 无弹性调度 | 平均写延迟、吞吐、ProxyServer 分配数 |
| One-Hop 读 | 传统 Proxy 读 | 读延迟 |
| TierTable 辅助读 | Pure One-Hop | 读延迟 |
| 24h 生产追踪 | — | 全面性能指标 |

### 核心实验结论

- **写路径**：动态 Blob 分配将热点负载比从 11.7%→4.5%，P99 延迟降 7.16%；跨集群 ProxyServer 快速再分配有效应对流量突增
- **读路径**：One-Hop 读在所有测试配置下一致降低延迟（最高减少 1197.68 μs）；TierTable 辅助读在活跃写入场景下进一步降低延迟（最高减少 1996.21 μs）
- **生产验证**：24h 追踪中写延迟最大仅 3.55ms，高吞吐下系统稳定运行

---

## 批判性思考

### 优点

1. **工业落地验证充分**：基于 ByteDance 真实生产数据，提供了详细的 24h 生产追踪，可信度高
2. **有状态池化的系统性设计**：从写调度（Blob 分配→ProxyServer 分配→Group 再平衡）到读优化（One-Hop→TierTable→Blacklist）形成完整闭环，各模块耦合恰当
3. **代价函数优雅**：Rack-Oriented 分配的 MCMF 模型将分散性和亲和性两个矛盾目标统一在一个代价函数中，工程可解且约束条件清晰
4. **渐进式直接路径读**：One-Hop + Client Cache + TierTable 的三层优化，既照顾了首次读的正确性，又最大化了后续读的性能

### 局限性

1. **SIGMOD Companion 定位**：论文属于 SIGMOD Industry/Companion 轨道，技术深度相对于正式 Research Paper 有限，部分实现细节（如 MCMF 的具体图构造参数调优）未完全展开
2. **TierTable 生命周期张力**：论文同时强调 TierTable 是读加速的关键，又明确不为读而延长 TierTable 存活，这一策略在写吞吐极高场景下可能导致 TierTable 命中率下降
3. **跨数据中心场景受限**：论文明确指出不同数据中心的 ProxyServer 服务不同用户群体、不可互换，池化仅在单数据中心内进行

### 潜在改进方向

1. **TierTable 智能 TTL**：根据读流量动态调整 TierTable 存活时间，在资源允许时适当延长以提升读命中率
2. **预测式 ProxyServer 分配**：结合历史流量模式提前预判突增，减少 ProxyServer 重分配时的瞬态延迟抖动
3. **跨数据中心冷路径 Proxy 复用**：在跨 DC 延迟可接受的场景下，探索有限度的跨 DC Proxy 资源借用

### 可复现性评估

- [ ] 代码开源（未开源）
- [ ] 预训练模型（不适用）
- [x] 实验设计描述完整
- [ ] 内部系统（ByteStore/ByteTier）无法获取

---

## 关联笔记

### 基于

- [[ByteStore]]: ByteDance 自研 DFS，CLAPS 的底层存储基础设施，提供 Blob 存储和 ChunkServer/MetaServer 接口
- [[ByteTier]]: 基于 ByteStore 的两阶段写机制层，引入 TierTable 和 WAL，是 Proxy 有状态化的根本原因

### 对比

- [[Fisc]]: 阿里云文件存储，使用共享内存优化转发路径，但仍需代理参与读
- [[DiSC]]: 多层 TCP 连接共享绕过代理，依赖全局同步序列，复杂度高

### 方法相关

- [[MCMF|最小费用最大流]]: Rack-Oriented ProxyServer 分配的核心优化算法
- [[WAL|Write-Ahead Log]]: ByteTier 写状态的核心载体，ProxyServer 有状态迁移必须处理 WAL 重加载
- [[TierTable]]: RW ProxyServer 的写缓冲结构，同时作为读加速组件
- [[SPFA]]: Bellman-Ford 的队列优化版本，用于 MCMF 中的最短增广路查找
- [[Proxy 资源池化|Proxy Pooling]]: 核心系统设计范式，取代孤立静态绑定实现弹性共享

### 硬件/基础设施相关

- [[ChunkServer]]: ByteStore 存储节点，直接存储数据 Chunk，One-Hop 读的最终目标
- [[MetaServer]]: ByteStore 元数据服务器，存储 Blob 位置信息和 WAL 关联状态

---

## 速查卡片

> [!summary] CLAPS (SIGMOD '26, ByteDance)
> - **核心**: 将有状态 ProxyServer 集群整合为共享弹性资源池，消除静态 Proxy-Storage 绑定的资源冗余
> - **写路径**: 动态 Blob 分配（排除 Top15% 过载节点）+ MCMF 机架感知分配 + 冻结 Group 优先迁移
> - **读路径**: One-Hop（Client Cache 直连 ChunkServer）+ TierTable 辅助读 + 双层黑名单容错
> - **结果**: P99 写延迟↓7.16%，One-Hop 读延迟↓最高 1197μs，TierTable 进一步↓最高 1996μs，生产验证 24h 稳定
> - **代码**: 未开源（ByteDance 内部系统）

---

*笔记创建时间: 2026-07-14*
