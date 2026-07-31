---
title: "Aurora PostgreSQL Limitless Database: Building a Highly Scalable OLTP Database"
method_name: "Aurora Limitless Database"
aliases: [Aurora Limitless Database, Aurora Limitless, Aurora PostgreSQL Limitless]
authors: [Dmitry Arkhangelskiy, Saikiran Avula, Sachit Batra, Jin Chen, Radwan Deeb, Alexey Gotsman, Upendra Gowda, Haritabh Gupta, Benoit Hudzia, Rishabh Jain, Kaumudi Kaushik, Aravind Kumar Kumar, Sergey Melnik, Saleem Mohideen, Sharique Muhammed, Davor Prugovecki, Sanjay Shanthakumar, Sagar Shedge, Anand Kumar Thakur, David Wein]
year: 2026
venue: SIGMOD Companion 2026
tags: [distributed-database, sharding, two-phase-commit, snapshot-isolation, postgresql, cloud-native, oltp, adaptive-scaling]
zotero_collection: SIGMOD/2026
image_source: local
created: 2026-07-31
---

# 论文笔记：Aurora PostgreSQL Limitless Database: Building a Highly Scalable OLTP Database

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | Amazon Web Services (Seattle, USA)；Alexey Gotsman 兼 IMDEA Software Institute (Madrid, Spain) |
| 日期 | 2026 年 5 月（SIGMOD Companion '26, Bengaluru, India） |
| DOI | https://doi.org/10.1145/3788853.3803089 |
| 页码 | pp. 175–187（13 页） |
| 对比基线 | [[Citus]] / [[Greenplum]] / [[Spanner]] / [[CockroachDB]] / [[YugabyteDB]] / [[Aurora DSQL]] / [[PolarDB]] |
| 生产状态 | 已在 AWS 生产运行 1 年以上，最大集群 64 shards + 32 routers |
| 基准 | [[HammerDB]]（[[TPC-C]] 衍生），12000 warehouses，1000 虚拟用户 |

---

## 一句话总结

> 通过 router/shard 分层 + [[Amazon Time Sync|时间同步服务]] 驱动的时间戳 MVCC 与非阻塞 [[Two-Phase Commit|2PC]]，把单机 PostgreSQL 的完整语义扩展到无中心协调者的水平分片集群，同时支持垂直+水平双维度自动伸缩。

---

## 核心贡献

1. **无中心事务管理器的写扩展**：用 [[Amazon Time Sync]] 提供的有界误差时钟替代 PostgreSQL 基于 xid 集合的快照，快照与提交时间戳都是标量，从而摆脱了 [[Greenplum]] 式的中心化 coordinator 瓶颈。
2. **Lead Shard 化的时间感知 2PC**：把 2PC 的事务状态持久化在一个被选中的 **lead shard**（而非无备机的 router）上，使 router 故障时的阻塞时间从「分钟级」降到「秒级」，实现了非阻塞的 2PC 变体。
3. **PostgreSQL 语义完整保留**：DML 与 DDL 都保持 PostgreSQL 语义，[[Repeatable Read]]（= [[Snapshot Isolation]]）与 [[Read Committed]] 两个隔离级别均被扩展到跨分片场景，并额外提供 [[External Consistency|外部一致性]]。
4. **混合式自适应伸缩**：垂直（[[Aurora Serverless V2]] 按 [[Aurora Capacity Unit|ACU]] 调容）与水平（[[Table Slice|table slices]] 迁移做 shard split / router 增加）两个维度组合，其中 shard split 靠 Aurora 的 [[Copy-on-Write Cloning|存储层写时复制克隆]] 卸载数据搬迁开销。
5. **时间点一致的集群级备份**：利用时间戳提交协议，让每个 backup 精确对应「commitTs ≤ t 的所有事务」，跨所有 shard 全局一致。
6. **分布式查询下推优化器**：借用 PostgreSQL 的 **partitioned table + [[Foreign Data Wrapper|FDW]]** 机制表达分片数据，从而复用 PostgreSQL planner 做 partition-wise join、部分聚合下推和 [[Partition Pruning|分区裁剪]]，并把单分片查询优化成「甜点路径」。

---

## 问题背景

### 要解决的问题

高吞吐云原生 OLTP 负载（金融实时交易、SaaS 多租户等）最终会撞上传统**单主库架构**的三重硬上限：

- 写吞吐上限（单个 primary 的 CPU/日志带宽）
- 存储容量上限
- 并发连接数上限

要突破就必须做水平分片，但应用层分片（application-level sharding）要求业务自己处理路由、跨分片事务和 schema 演进，运维负担极重。

### 现有方法的局限

| 系统 | 局限 |
|------|------|
| [[Citus]] | 有 2PC 保证 ACD，但**不为分布式事务提供 [[Snapshot Isolation]]**，用户暴露在大量事务异常之下 |
| [[Greenplum]] | 提供 SI，但靠**中心化 coordinator** 分配分布式事务 ID，负载升高即成瓶颈；还需维护 local↔distributed 事务映射做可见性判断，开销与复杂度双高 |
| [[CockroachDB]] | 支持 serializability 与 RC，但**不支持 SI**；用 [[Hybrid Logical Clock]] (HLC) 定 commit 时间戳，一般**不保证外部一致性** |
| [[YugabyteDB]] | 支持 serializability 与 SI，同样基于 HLC；HLC 在无 TrueTime / Time Sync 时须**保守地等待数百毫秒**的最大时钟偏移 |
| [[Vitess]] / [[Apache ShardingSphere]] | 提供水平扩展，但运维管理复杂 |
| [[Azure Database for PostgreSQL]] / [[Google Cloud SQL]] | 只做单实例垂直扩展 |
| [[PolarDB]] | commit 时间戳来自 RDMA timestamp oracle 或 HLC，后者以放松一致性换低延迟 |

### 本文的动机

作者的判断是：**「有界误差的高精度时钟」这一基础设施已经足够便宜可靠**——[[Amazon Time Sync]] 在所有 AWS region 的 clock error bound (CEB) 通常 <1ms，部分 region 已降至低双位数微秒级。

于是可以走 [[Clock-SI]] 的路线：用时间戳而非中心化 ID 分配器来定义快照与提交顺序，把跨分片一致性问题局部化到「每个 shard 各自读时钟 + 取 max」。同时因为目标场景是**跨 AZ 而非跨 region**，可以直接复用 Aurora 已有的 3-AZ 存储层做持久化，**完全不需要额外引入 Paxos / Raft 共识协议**——这是相对于地理分布式数据库的一个关键简化。

---

## 数据模型

Aurora Limitless 保持 `CREATE TABLE` 语法**完全不变**，通过 session 参数 `limitless_create_table_mode` 声明表类型，共三种：

| 表类型 | 放置方式 | 适用场景 |
|--------|----------|----------|
| **Sharded table** | 按 **shard key**（`limitless_create_table_shard_key`）哈希水平分片到所有 shard | 主体大表 |
| **Reference table** | **全量复制**到每个 shard | 小且更新不频繁、需频繁与分片表 join 的维表 |
| **Standard table** | 只放在配置指定的**单个 shard** | 无需分片的数据；也是从传统 PostgreSQL 导入数据的落脚点，后续可转成 sharded/reference |

**[[Data Co-location|共置 (co-location)]]**：共享同一 shard key 的表可通过 `limitless_create_table_collocate_with` 声明共置，保证同一 shard-key 值的数据落在同一 shard，从而让 join 可以在 shard 本地完成。

### Figure 1: 三种表类型的示例 schema

```sql
SET rds_aurora.limitless_create_table_mode = 'sharded';
SET rds_aurora.limitless_create_table_shard_key = '{"cust_id"}';
CREATE TABLE customers (
    cust_id INT PRIMARY KEY NOT NULL,
    name TEXT,
    email VARCHAR(100) );

SET rds_aurora.limitless_create_table_collocate_with = 'customers';
CREATE TABLE orders (
    order_id INT NOT NULL,
    cust_id INT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    tax_rate_id INT,
    PRIMARY KEY (order_id, cust_id) );

SET rds_aurora.limitless_create_table_mode = 'reference';
CREATE TABLE tax_rates (
    tax_rate_id INT PRIMARY KEY NOT NULL,
    state TEXT,
    country TEXT NOT NULL,
    tax_rate DECIMAL(5,4) NOT NULL );

SET rds_aurora.limitless_create_table_mode = 'standard';
...
```

**说明**：`customers` 按 `cust_id` 分片；`orders` 与 `customers` 共置（因此同一客户的客户信息与订单在同一 shard，join 可本地完成）；`tax_rates` 因为小且少变而声明为 reference 表，与其它表 join 时更可能退化为单分片查询。

---

## 方法详解

### 系统架构

Aurora Limitless 是**多层架构**，控制操作与数据处理分离，构建在 [[Amazon Aurora]] 的存算分离存储与 [[Aurora Serverless V2]] 之上。一个 Limitless 数据库中的 router + shard 全体称为一个 **shard group**。

- **存储层**：[[Aurora Distributed Storage]]
  - 数据卷切成 segment，每次写复制到 **6 个存储节点（3 AZ × 2）**
  - 可容忍「整个 AZ 故障 + 另一 AZ 中一个副本故障」而不丢数据；整 AZ 故障下仍可写
  - 自动检测并修复失效 segment，容量弹性伸缩
- **数据平面**：两个**可独立伸缩**的 fleet，每个 router/shard 本身都是一个带独立存储卷的 PostgreSQL 集群
  - **Shards**：拥有真实数据分片；接收 router 下发的 plan fragment 并在本地数据上执行
  - **Routers**：**不存数据**，只维护元数据 —— 集群拓扑（router/shard 集合与健康状态）、schema 定义、数据放置映射（哪个 shard-key range 在哪个 shard）。router 是 schema 与放置映射的**权威来源**，而拓扑信息由控制平面的元数据服务同步下来
- **控制平面**：
  - **Cluster-management service**：shard/router 生命周期、拓扑权威注册表、拓扑变更传播、节点 provision/deprovision 工作流（重度依赖 Aurora Serverless V2 基础设施）
  - **Monitoring service**：每节点本地 agent 监控健康并在需要时触发 failover；跨节点服务检测网络分区；指标进 Amazon CloudWatch
  - **Heat-management service**：驱动自动伸缩（§4）
  - **Backup-and-restore service**（§6）

### Figure 2: Aurora Limitless 数据库架构

![[AuroraLimitless_fig2_architecture.png]]

**说明**：自上而下为 Routers 层（接客户端流量）→ Shards 层（持有数据分片）→ Aurora distributed storage（3-AZ durability），右侧为独立的 Control Plane。Router 与 Shard 之间双向通信下发计划片段/回传结果，两层各自的节点都直接挂在同一套 Aurora 分布式存储上。

### 连接管理与会话多路复用

- 客户端连接通过 **DNS-based load balancing** 分散到 router fleet；一旦建立，连接在整个 session 生命周期内**绑定到某个特定 router**
- 每个 (router, shard) 对之间建立**多条连接**；router 的 connection manager 把客户端 session **多路复用**到这些 shard 连接上，并按负载动态分配容量
- 多路复用粒度是 **事务级**：事务结束后，它占用的 shard 连接即可被该 router 上其它事务复用
- 会话状态（认证、role、变量）通过 **session-context passing** 跨事务与跨 shard 边界保留

### 高可用

| 组件 | 失效处理 |
|------|----------|
| **Router** | 所有 router 同质，经单一 endpoint 的 DNS 解析访问。某个 router 不可用时，其它 router 通过 DNS 解析接手流量，同时故障 router 恢复或新 router 加入。**因此 router 没有专用 standby，以此省成本** |
| **Shard** | shard 故障会让其数据在重新 provision 前不可用，因此 shard **可以有 standby**（0～2 个，客户可配，分布在不同 AZ）。故障时 standby 接管其存储卷与负载 |

注：shard standby 的容量总是与对应 primary shard 一致，且**不计入**客户设置的总容量预算。

---

## 自适应伸缩

Aurora Limitless 实现**二维自动伸缩**：router 与 shard 都既可垂直（调容量）也可水平（加实例/分数据）伸缩。

容量单位是 **[[Aurora Capacity Unit|ACU]]**：1 ACU ≈ 2 GB 内存 + 对应的 CPU 与网络资源。客户在创建 shard group 时设定**总的最小/最大计算容量**，初始节点数由**最大容量**推导——容量越高，router 与 shard 越多。

### Table 1: 由最大容量决定的初始节点数与默认最小容量映射（部分）

| Total nodes | Routers | Shards | Default min ACUs | Max ACU range |
|-------------|---------|--------|------------------|---------------|
| 4  | 2 | 2  | 16  | 16–400 |
| 12 | 4 | 8  | 48  | 1101–1200 |
| 18 | 6 | 12 | 72  | 1701–1800 |
| 24 | 8 | 16 | 96  | 2301–6144 |

**说明**：例如客户把总 max ACU 设为 1200，系统会初始创建 **4 routers + 8 shards**。注意表中的容量与节点数**不含** shard standby（standby 容量恒等于对应 primary，且不计入总预算）。生产中观察到的最大 shard group 是 **32 routers + 64 shards**，而**最常用配置是 4 routers + 8 shards**。

### 垂直伸缩

每个节点有自己的 min/max ACU。初始时 ACU 预算在 shard group 全体成员间**均分**，之后系统周期性地**按各节点实际消耗的 ACU 比例**重新分配配额。设 shard group 有 $n$ 个节点，节点 $i$ 的动态配额为：

### 公式1: [[Proportional ACU Allocation|动态最小 ACU 配额]]

$$
\text{dynamicMinACU}_i = \text{shardGroupMinACU} \cdot \frac{\text{consumedACU}_i}{\sum_{i=1}^{n} \text{consumedACU}_i}
$$

**含义**：把 shard group 的最小容量预算，按各节点当前实际消耗量的占比切分给每个节点，作为该节点的 scale-down 下界。

**符号说明**：
- $n$：shard group 中的节点总数（router + primary shard）
- $\text{shardGroupMinACU}$：客户设定的 shard group 总最小容量
- $\text{consumedACU}_i$：节点 $i$ 当前实际消耗的 ACU

### 公式2: [[Proportional ACU Allocation|动态最大 ACU 配额]]

$$
\text{dynamicMaxACU}_i = \text{shardGroupMaxACU} \cdot \frac{\text{consumedACU}_i}{\sum_{i=1}^{n} \text{consumedACU}_i}
$$

**含义**：同理把总最大容量按实际消耗占比分给各节点，作为其 scale-up 上界。

**符号说明**：
- $\text{shardGroupMaxACU}$：客户设定的 shard group 总最大容量
- 其余符号同公式 1

> **设计依据**：消耗更多容量的节点通常也表现出更高的增长需求，需要更快地扩容，所以按消耗比例分配配额是合理的。

**router 与 shard 的伸缩策略差异**：

| 维度 | Shard | Router |
|------|-------|--------|
| 负载特征 | 执行复杂查询、从存储读大量数据 → **高 compute + 高 buffer cache** 占用 | 大部分数据从 shard 读取，处理分布式查询时**占用大量内存**存放中间数据 |
| 伸缩触发信号 | 更关注 **compute 使用率** | 更关注 **memory 使用率** |
| 内存分配策略 | 分配更多 **buffer cache** 内存 | 分配更多 **heap** 内存 |

**非对称伸缩速率**（同 [[Aurora Serverless V2]]）：scale-up 在负载尖峰时**快速触发**以保证响应性；scale-down 更**保守**，须在低利用率持续较长时间后才触发，以防资源抖动（thrashing）。

### 水平伸缩

客户可以通过系统提供的函数发起**shard split（一分为二）**或**新增 router**，这些函数启动异步 workflow，另有函数查询完成状态。系统向客户暴露 router/shard 的 ACU 与存储消耗监控信息，供客户判断何时需要水平伸缩；客户也可以授权系统做**自动 shard split**——此时 heat-management service 会监控 shard 的 ACU 与存储指标，在**垂直扩容之后指标仍超过预设阈值**时触发分裂。

**容量管理**：新增节点时，其容量预算从 shard group 现有成员**未消耗的容量**中划出；若余量不足则**拒绝客户请求**。系统会依据现有节点的数据估算新节点的初始 ACU 消耗，然后按公式 1、2 重新分配全体成员的最大容量配额。

#### [[Table Slice|Table Slices]]：分裂的基本单元

shard split 的基础是一层更细粒度的分区抽象 **table slices**：

- 每个 shard 为每张分片表持有一段 shard key 的哈希范围，Limitless 把该范围进一步细分为 slices，**典型为每张表 512 个 slice** 分布在各 shard 上
- shard 在内部把自己那部分分片表表示为一张 **PostgreSQL 分区表**，分区即对应 slice
  - ⚠️ 因此分片表整体上存在**两层 PostgreSQL 式分区**：router 层按 shard 分区，shard 层按 slice 分区
- slice 是 shard split 时的**最小迁移单元**
- **共置表的对应 slice 一起迁移**，从而保留 join 优化
- **standard table 永不分裂**，始终留在同一 shard

分裂点的选择：客户手动发起时，源 shard 上**一半的 slice** 迁到新 shard（哈希范围居中对半切）；自动分裂时，则依据 heat-monitoring service 提供的**热点 slice 信息**决定切分点。

#### Shard-split 工作流（4 阶段）

- **Phase 1: 存储层克隆 (Storage-level cloning)**
  - 朴素做法是从源 shard 读数据再写到新 shard，但这会给**本已过载的 shard 再加负载**
  - 因此改用 Aurora 的 **[[Copy-on-Write Cloning|写时复制克隆]]** 机制：克隆源 shard 的存储卷，数据仅在发生修改时才真正复制
  - 把克隆工作**卸载到存储层**，使得即使 shard 已满载运行也能完成分裂
- **Phase 2: Redo 日志回放 (Redo-log replay)**
  - 克隆卷需要时间，期间源 shard 仍在写入，克隆完成时副本可能已过时
  - 新 shard 开始应用源 shard 的 **redo log** 中的待处理更新，追赶源 shard
- **Phase 3: 切换 (Switchover)**
  - 新 shard**接近追平**后启动切换
  - 在**所有 router 与 shard 上锁住**待迁移 slice 所属的表，阻塞对它们的新写入；若因既有事务持有冲突锁而无法立即获取，则**终止这些事务**强制释放锁
  - 加锁等效于**冻结**受影响表的状态，源 shard 不再为它们产生新的 redo 记录，新 shard 得以最终完成回放
  - 之后：router 更新 slice→shard 映射，把迁移的 slice 指向新 shard；源 shard 用 `DETACH PARTITION` 解链已迁走的 slice；新 shard 解链那些克隆来但不由自己持有的 slice
  - 释放切换期获取的锁，后续针对迁移 slice 的事务自动路由到新 shard
- **Phase 4: 清理 (Clean-up)**
  - 切换后两个 shard 都临时持有已不属于自己的 slice（源 shard 持有已迁走的，新 shard 持有归属源 shard 的）
  - 两者在后台删除不再映射到自己的 slice

> **客户影响面**：整个分裂过程中客户负载基本正常运行，**唯一的影响窗口是 switchover 阶段**——此时 DDL 操作与对迁移中 slice 的更新被临时阻塞，且部分进行中的事务可能被终止。

**Router 新增**：流程与 shard split 类似——克隆一个既有 router 成为新实例，通过 cluster-management service 加入拓扑，最后注册到 DNS，之后客户即可连接到新 router。

**水平 vs 垂直**：两者是互补的两个伸缩维度，客户可结合自身负载特征组合使用，以**最大化性能**或**优化性价比**。

---

## 并发控制与提交协议

Aurora Limitless 提供 PostgreSQL 中最常用的两个隔离级别：**[[Repeatable Read]] (RR)**（在 PostgreSQL 中即 [[Snapshot Isolation|SI]]）与 **[[Read Committed]] (RC)**。核心 SI 实现思路接近 [[Clock-SI]]。

### 时间戳来源：Amazon Time Sync

PostgreSQL 原生快照的问题：

- 每个做修改的事务被分配一个来自**全局计数器**的唯一 **xid**
- 为事务 $T$ 计算快照时，需记录 `xmin`（当前运行中事务的最小 id）、`xmax`（最小的尚未分配 id）、`xip_list`（介于两者之间的所有运行中事务 id）
- $T$ 的可见规则：除自身写入外，只看到快照拍摄时已提交的事务，即所有 `xid < xmin` 的已提交事务，以及 `xid < xmax` 且 `xid ∉ xip_list` 的已提交事务
- **两大缺陷**：① 需记录所有运行中事务，快照计算是 **CPU 密集**的，是高吞吐 PostgreSQL 上众所周知的争用源；② 在分布式数据库中，**单个节点无法知晓所有运行中事务**

Aurora Limitless 改用 **[[Amazon Time Sync]]** 生成的时间戳定义快照：

- 该服务在每个 region 用一组冗余的**卫星连接原子钟**提供**有界时钟不确定性**
- 函数 `now()` 返回当前时间 $t$ 以及 **clock error bound (CEB)**，真实时间保证落在 $[\text{earliest}, \text{latest}]$ 区间内，其中 $\text{earliest} = t - \text{CEB}$，$\text{latest} = t + \text{CEB}$
- **典型 CEB 在所有 AWS region 都 <1 毫秒**，部分 region 已降到**低双位数微秒**
- 该 API 由开源的 **[[ClockBound]] daemon** 提供，所有 AWS 客户可用

时间戳分配：

- 事务执行**第一条查询**时，其 router 赋予它 **snapshot timestamp** $\text{startTs} = \texttt{now().latest}$（取当前 Time Sync 区间的**上界**）
- 该时间戳**捎带（piggyback）在 router 发往 shard 的每条查询**上，决定该读什么数据
- 事务提交时被赋予 **commit timestamp** $\text{commitTs}$，同样由 Time Sync 计算得出

### Property 1（SI 可见性规范）

> 给定两个不同事务 $T$ 与 $T'$，$T'$ 的修改被包含在 $T$ 所见的快照中 **当且仅当** $T'.\text{commitTs} \le T.\text{startTs}$。

后续所有机制的目的都是保证这条性质成立。

### 公式3: [[Timestamp-Based MVCC|基于时间戳的版本可见性判断]]

PostgreSQL 为每行维护多个版本，每个版本关联创建它的事务的 `xmin` 与创建其下一版本的事务的 `xmax`（无下一版本时 `xmax` 无效）。原生 PostgreSQL 通过把版本的 `xmin`/`xmax` 与 $T$ 的 xid 快照比较来判可见性；Aurora Limitless 改为用时间戳：一个已提交的行版本对事务 $T$ 可见当且仅当

$$
\text{xmin.commitTs} \le T.\text{startTs} < \text{xmax.commitTs}
$$

**含义**：创建该版本的事务必须在 $T$ 的快照时刻之前提交，且覆盖该版本的事务必须在 $T$ 的快照时刻之后才提交。这直接蕴含了 Property 1：一旦 $T'$ 提交，其改动即对所有 $T.\text{startTs} \ge T'.\text{commitTs}$ 的事务可见。

**符号说明**：
- $\text{xmin.commitTs}$：创建该行版本的事务的提交时间戳
- $\text{xmax.commitTs}$：创建该行下一版本的事务的提交时间戳；**若 `xmax` 无效则省略右侧判断**
- $T.\text{startTs}$：读事务 $T$ 的快照时间戳

**实现要点**：为了**不改动 PostgreSQL 行格式**，shard 单独维护一张「事务 id → commit 时间戳」的映射，并把它集成进 PostgreSQL 现有的 **commit log** 数据结构（原本用于记录哪些事务已提交）。

### 写冲突检测

SI 要求：事务 $T$ 只有在其执行期间没有别的事务写过相同行时才能提交。Aurora Limitless 的做法与 PostgreSQL 一致：

1. $T$ 修改某行时，在对应 shard 上获取该行的**排他锁**，并**持有至提交**，从而阻止并发修改
2. 拿到锁后，$T$ 再检查该行**当前存在的所有版本是否仍被自己的快照覆盖**——若存在更新的版本，则**事务中止**
3. 检查通过后，$T$ 在对应 shard 上创建该行的新版本

### 时间感知的两阶段提交 (Time-Aware 2PC)

多分片事务用 **[[Two-Phase Commit|2PC]]** 提交，目标是让所有参与者对事务结局、以及（若提交）对**提交时间戳**达成一致。

**关键设计难点**：router 是 2PC coordinator 的自然选择，但为省成本 router **没有专用 standby**（§3.4），host 故障后替换 router 需要**数分钟**，这段时间 2PC 会被阻塞。

**解法 —— [[Lead Shard]]**：把事务状态**持久化在一个被指定的 lead shard 上而非 router**。lead shard 可以有 standby，于是替换与状态恢复是**秒级**而非分钟级。

### Figure 3: 带 lead shard 的 2PC 协议

![[AuroraLimitless_fig3_2pc_lead_shard.png]]

**说明**：时序图展示 Router、Shard 1 (lead)、Shard 2、Shard 3 四方交互。Client commit 到达后 router 向非 lead 的参与 shard 广播 prepare（携带 lead shard id），收齐 ACK 后向 lead shard 发 commit request，lead shard 决定 commit time 并本地提交、回 ACK，router 随即向客户端确认提交，**然后才异步**向其余 shard 发送 commit prepared 请求。

**无故障流程**：

1. 收到提交请求后，router 从**本事务更新过的 shard 中**选一个作为 **lead shard**，向**其余所有被更新的 shard** 发送 `PREPARE TRANSACTION` 命令（携带 lead shard 的标识）
2. 每个 shard 用 Time Sync 的 `now().latest` 计算自己的 **prepare timestamp**（实际计算更微妙，见下文 HLC），然后把 prepare 信息与 lead shard 标识**持久化到自己的存储卷**，并把时间戳回给 router
3. 收齐所有回复后，router 把**所有 prepare 时间戳的最大值**发给 lead shard
4. lead shard 把 **commit 时间戳**算作「router 发来的时间戳」与「自己的时间戳提案（算法同 prepare timestamp）」两者的**最大值**，持久化该 commit 时间戳，本地提交事务，并向 router 发确认
5. router 收到确认后**先通知客户端**，再向其余参与 shard 发送带 commit 时间戳的 `COMMIT PREPARED` 命令，它们随之提交

**异常处理**：

- 若某 shard 无法 prepare（例如写存储失败），通知 router，router 向所有 shard 发 `ROLLBACK PREPARED`
- 若 **router 在 2PC 期间故障**，协议依赖 lead shard 一致地解决进行中的事务，按进度分三种情况：
  - **尚未在任何 shard 上 prepare** 的事务 → **中止**（没有持久状态记录）
  - **已 prepare 但未在部分 shard 上 commit** 的事务 → **以 lead shard 的状态为权威**，其余 shard 向 lead shard 查询事务状态
    - lead shard 已提交 → 所有 shard 都提交
    - 否则 → 中止（典型情形：router 在请求 lead shard 提交之前就故障了）

**保证**：一旦事务的提交被确认给客户端，其效果就**持久可见**，无论后续发生什么故障；且所有参与者对提交时间达成一致，该时间**不低于所有 prepare 时间戳的最大值**。

**关键优化 —— 绕过 2PC**（生产负载中占主导的两类事务）：

- **只读事务**：router **立即提交**，因为它们在执行期间观察到的就是一致数据
- **单分片更新事务**：router 把提交**转发给被更新的那个 shard**，由它本地管理提交流程并自行选定提交时间。⚠️ **即使该事务从多个 shard 读过数据，此优化仍然适用**（只要写只落在一个 shard）

### 从 shard 读数据的三类一致性难题

事务 $T$ 读某 shard 时，除了比较当前已提交版本的 `xmin`/`xmax` 与自己的 `startTs`（§公式 3），还必须考虑该 shard **未来可能提交**的事务所写的版本。

#### 难题 1：时钟偏移 (Clock Skew) —— 用 HLC 免等待

**问题场景**：router 与 shard 之间的时钟偏移可能让读事务的快照时间戳**领先于** shard 的本地时间。例如 $T$ 的 $\text{startTs} = 100$，而目标 shard 的本地 Time Sync 区间是 $[20, 40]$。此时 $T$ 不能直接读该 shard 已提交的数据——因为读完之后，该 shard 完全可能提交一个 $\text{commitTs} = 60$ 的事务 $T'$，而 $T'$ 本应落在 $T$ 的快照内（Property 1），于是 $T$ 就漏读了。

**传统做法**（[[Clock-SI]]）：**延迟** $T$ 的读，直到 shard 本地时间推进过其快照时间戳。

**Aurora Limitless 做法**：用 **[[Hybrid Logical Clock|HLC]]** 让 $T$ **立即读**：

### 公式4: [[Hybrid Logical Clock|读操作推进 shard 逻辑时钟]]

每个 shard 维护一个逻辑时钟 $C$，每次被事务 $T$ 读取时推进为

$$
C \leftarrow \max\{C,\ T.\text{startTs} + 1\}
$$

**含义**：把 shard 的逻辑时钟强行抬到读者快照时间戳之上，使该 shard 此后提交的任何事务的时间戳都必然大于 $T.\text{startTs}$，于是 $T$ 无需等待即可安全读取当前已提交数据。

**符号说明**：
- $C$：shard 本地维护的混合逻辑时钟
- $T.\text{startTs}$：正在读取该 shard 的事务的快照时间戳

### 公式5: [[Hybrid Logical Clock|HLC 化的 prepare/commit 时间戳]]

当 shard 被要求 prepare 分布式事务 $T'$ 时，其 prepare 时间戳计算为

$$
T'.\text{prepareTs} = \max\{C,\ \texttt{now().latest}\}
$$

**含义**：prepare 时间戳同时不低于「本地逻辑时钟」与「Time Sync 上界」。**单分片事务的提交时间也用同一公式计算**。结合「分布式事务的 commit 时间 = 各 shard prepare 时间戳的最大值」（§5.4），即可推出：**一旦 $T$ 读过某 shard，该 shard 此后提交的任何事务的 commitTs 都 $> T.\text{startTs}$**，这正是免去延迟又不破坏一致性的原因。

**符号说明**：
- $C$：该 shard 的混合逻辑时钟当前值
- $\texttt{now().latest}$：Time Sync 返回区间的上界

#### 难题 2：已 prepare 但未定论的事务

**问题场景**：$T$ 想读 shard $S$ 上的某行，而一个更新该行的分布式事务 $T'$ 已在 $S$ 上 prepare，且其 prepare 时间戳 $< T.\text{startTs}$。此时 $S$ **无法判断** $T'$ 最终的 commit 时间戳会落在 $T.\text{startTs}$ 之下还是之上——若落在之下而 $S$ 又放 $T$ 读了旧值，就违反了 Property 1。

**解法**：$S$ 向 $T'$ 的 **lead shard 发起询问**（inquiry），询问中**附带 $T$ 的快照时间戳**：

- 若 $T'$ **已提交** → lead shard 回复其 commit 时间戳，$S$ 据此判断 $T$ 是否该看到 $T'$ 的修改
- 否则 → lead shard 把自己的逻辑时钟推进为 $\max\{C,\ T.\text{startTs}+1\}$ 后回复 $S$。这保证了 $T'$ 最终拿到的 commit 时间戳**必然高于** $T.\text{startTs}$，因此其修改对 $T$ **不可见**

#### 难题 3：处于 COMMITTING 状态的事务

**问题场景**：事务在其 lead shard（分布式事务）或被下推的那个 shard（单分片事务）上提交时，存在一个**短暂窗口**——已经取得 commit 时间但尚未真正提交（例如日志记录还在刷向存储）。这段时间事务被标记为特殊的 **COMMITTING** 状态。

由于 COMMITTING 的事务**仍可能最终中止**（比如写存储失败），其更新**不能被安全读取**。

**解法**：$T$ 读某行时若该 shard 正在 COMMITTING 另一个更新了该行的事务 $T'$，则：若 $T'.\text{commitTs} \le T.\text{startTs}$，$T$ **必须等待** $T'$ 提交或中止之后再读该行。

### 保持实时序 (Real-Time Order) 与外部一致性

Aurora Limitless 实现的是**强化版 SI**——尊重事务间的实时顺序：

> 若事务 $T_2$ 在（非只读）事务 $T_1$ 向客户端返回**之后**才开始，则 $T_2.\text{startTs} > T_1.\text{commitTs}$。

因此 $T_2$ **必然看到** $T_1$ 的修改，**无论两者从哪个节点发起**。这直接改善客户体验：事务总能看到最新数据。

> 📌 论文脚注指出，实际保证的是更强的 **[[External Consistency|外部一致性]]**：若 $T_2$ 在 $T_1$ 结束**之后**开始提交，则 $T_2.\text{commitTs} > T_1.\text{commitTs}$。

### 公式6: [[Commit Wait|Commit Wait 条件]]

提交做了修改的事务 $T_1$ 时，lead shard 确定 $T_1.\text{commitTs}$ 后**不立即**向 router 回确认，而是先等待直到

$$
\texttt{now().earliest} > T_1.\text{commitTs}
$$

**含义**：这个 **commit wait**（源自 [[Spanner]]）确保「事务对客户端可见的时刻」在**真实时间**上一定晚于其提交时间戳，从而保持实时序。

**符号说明**：
- $\texttt{now().earliest} = t - \text{CEB}$：Time Sync 区间的**下界**
- $T_1.\text{commitTs}$：$T_1$ 的提交时间戳

**正确性论证**（论文给出的三步推导）：

- 令 $t_1$ 为 $T_1$ **退出 commit wait 的真实时间**。由于 $t_1$ 必落在此刻 Time Sync 返回的区间内，有
  $$
  t_1 \ge \texttt{now().earliest} > T_1.\text{commitTs}
  $$
- 令 $t_2$ 为 $T_2$ **计算其快照的真实时间**。同理 $t_2$ 落在此刻的区间内，而 $T_2.\text{startTs} = \texttt{now().latest}$，故
  $$
  T_2.\text{startTs} = \texttt{now().latest} \ge t_2
  $$
- 若 $T_2$ 在 $T_1$ 结束**之后**开始，则 $t_2 > t_1$，于是
  $$
  T_2.\text{startTs} \ge t_2 > t_1 > T_1.\text{commitTs}
  $$
  即所需结论成立。

**延迟优化**：lead shard 把 commit wait 与**写 commit log 到存储并行执行**。由于存储写延迟通常**超过** Time Sync 误差界（典型 <1ms），这个等待**极少真正加到关键路径上**。

### DDL 语句

DDL 遵循 PostgreSQL 语义，特别是**多条 DDL 与 DML 可以打包进单个事务**以保证 ACID。

- 大多数 DDL 的执行、以及其所在事务的提交，**不只涉及存放相关表的 shard，还涉及所有 router**——这样才能保证 DDL 在**整个集群范围内原子可见**
- PostgreSQL 的 DDL 常需获取重量级锁以阻止并发 DML（例如 `ALTER TABLE` 一般会取 `ACCESS EXCLUSIVE` 锁）。在 Limitless 中，这类锁需要在**所有存放该表的 shard 以及所有 router 上**都获取
- 随后其所在事务通过 **§5.4 2PC 的一个变体**提交，该变体**额外把所有 router 作为参与者**
- 举例：Figure 1 中的 `CREATE TABLE customers` 会在各 shard 上创建该表的分区，同时把它加入各 router 的 schema，这个变更对并发执行的事务**原子可见**

**全局序列 (global sequence) 的特殊实现**：

1. 客户端提交 `CREATE SEQUENCE` 时，对应 router 先挑一个 shard，请求它创建全局序列对象用于**持久化**
2. router 随后向所有其它 router 广播初始化新序列的命令，为不同 router 分配**互不相交的号段 (disjoint ranges)**
3. 之后每个 router **无需联系 shard**即可从自己预分配的号段中发号，显著提升性能
4. 只有当自己的号段用尽时，router 才去联系那个存放全局序列的专用 shard 索取新号段

### 分布式死锁管理

Limitless 中的死锁可能跨越多个 router 与 shard：

- 客户可设置一个**超时**，等待资源超过该时长的事务即**怀疑存在分布式死锁**
- 此时系统选出一个 router，负责从各 shard 收集 **waits-for 图**，并在它们的**并集**中检测环
- 一旦发现环，该 router **随机选一个事务作为受害者**，向对应节点发命令中止它

**DDL 死锁的预防**：由于 DDL 可能需要在多个节点上加锁，为避免因**联系节点的顺序不同**而在 DDL 之间产生死锁，DDL 语句**总是先尝试在一个特定的（distinguished）节点上加锁**：

- 若该节点加锁**成功** → 再**并行**向其余所有节点发送加锁请求
- 若**失败** → **中止事务**

> 代价：DDL 执行时多一个 roundtrip；收益：规避了一类死锁。

### Read Committed 的实现

RC 与 RR 的实现基本相同，仅两点差异：

| 维度 | Repeatable Read (RR/SI) | Read Committed (RC) |
|------|-------------------------|---------------------|
| **快照时间戳** | 事务内**所有语句共用同一个** `startTs` | **每条语句都重新**用 `now().latest` 计算一个新的快照时间戳；读取结果 = 该时间戳决定的快照 + 事务自身已有的写入。因此事务执行过程中，其语句看到**越来越新**的数据库快照 |
| **写冲突检测**（§5.3） | 开启：若在事务开始与获取锁之间有别的事务改过该行，则**中止** | **关闭**：仍然获取行排他锁并持有至提交，但**不检查**该行是否被别的事务改过，事务照常继续 |

---

## 故障切换、备份与恢复

### Shard failover 与读租约 (read leases)

**基础机制**：Aurora 存储对给定卷**同一时刻只接受一个** Aurora PostgreSQL 实例的写入。Limitless 利用这一性质规避 **split-brain**：某 shard 被怀疑故障时，只有它的替代实例能写对应卷；若原实例其实还活着（即成为 **zombie**），它会在写入失败、意识到自己已被替换后**自行退出**。

**残留问题**：即使有上述保护，仍需防止 zombie shard **提供陈旧读**——它不知道自己被替换之后发生的写入，若允许它服务陈旧数据就会破坏**外部一致性**（§5.6）。

**解法 —— [[Lease|租约]]**：

- 每个 shard 持有一个 lease，通过**成功写入存储**来建立与续约
- 每次写入都**打上 shard 当前的时钟值 $t$**；写入成功即授予该 shard 一个**到 $t + \text{TTL}$ 为止**的租约（TTL 是租约时长参数）
- 该租约允许 shard 服务**快照时间 $\le t + \text{TTL}$** 的读
- 若 shard 多次尝试后仍**无法续约**，它就断定自己是 zombie 并**自我终止**

### 公式7: [[Lease|Failover 后的等待条件]]

Failover 到新实例后，该实例首先从 Aurora 存储卷中**恢复最后一次被授予的租约的起始时间** $t_{\text{last}}$，然后等待直到 Time Sync 返回

$$
\texttt{now().latest} > t_{\text{last}} + \text{TTL}
$$

之后才获取新租约并**开始处理事务**。

**含义**：确保新实例开始服务的时刻已晚于旧租约的失效时刻，使得 zombie 实例可能服务的任何读都不会漏掉它本应看到的数据。

**符号说明**：
- $t_{\text{last}}$：最后一次授予租约时打在写入上的时钟值
- $\text{TTL}$：租约时长参数
- $\texttt{now().latest}$：Time Sync 区间上界

**实践取值**：TTL **设得小于典型 failover 时间**，因此新实例启动时旧租约通常**已经过期**，多数情况下**无需额外等待**。

**协议正确性的理由**：shard 的 prepare 时间戳由 `now().latest` 计算，且最终 commit 时间戳保证**不低于**该 prepare 时间戳。因此上述等待结束后，新实例提交的任何事务的 commit 时间戳都 $> t_{\text{last}} + \text{TTL}$。这就保证了 zombie 实例**能够服务的那些读全都是有效的**——它们只可能漏掉本来就不该看到的事务。

### 备份与恢复协议

所有 Limitless 数据都存放在 Aurora 存储系统中，本身已有高持久性（§3.1）。客户可**额外为整个 shard group（含所有 shard 与 router）创建备份**并用于恢复数据库。

**与时间戳提交协议的集成**：

- 每个备份关联一个 Time Sync 时间戳 $t$，**精确反映所有 $\text{commitTs} \le t$ 的事务**，且**与它们在哪个 shard 上提交无关**
- 实现方式：节点写向 Aurora 存储的**每条日志记录**都打上事务提交协议为该记录关联的那个时间戳——特别地，事务的 commit 记录关联其 commit 时间戳
- 创建时间戳为 $t$ 的备份时，只把所有 shard-group 成员存储卷中**时间戳 $\le t$ 的日志条目**纳入备份
- 恢复时加载备份中的所有卷，并从该状态重启系统

**备份中的 prepare 悬挂事务**：在时间戳 $t$ 创建备份时，一个 $\text{commitTs} = t' \le t$ 的分布式事务**可能在部分 shard 上已提交、在其它 shard 上仅 prepare**。恢复后系统会被还原到**同样的状态**，那些处于 prepare 状态的 shard 按处理 router 故障的方式解决——即**向 lead shard 查询该事务的状态**（§5.4）。

---

## 查询处理与优化

### 表的内部表示

查询执行由 router 驱动：router 构造查询计划并编排其在各 shard 上的执行。三种表类型都被映射到 **PostgreSQL 已有的机制**上：

| 表类型 | Router 侧表示 |
|--------|---------------|
| **Sharded** | **PostgreSQL 分区表**，分区方式**镜像哈希空间在各 shard 间的划分**；每个分区是一张指向对应 shard 上实际数据的 **PostgreSQL foreign table** |
| **Reference** | 也表示为 foreign table；实现修改了 PostgreSQL 查询处理以尊重其「全 shard 复制」这一非标准放置方式 |
| **Standard** | 表示为指向**单个 shard** 的 foreign table |

- 依托 PostgreSQL 的 **[[Foreign Data Wrapper|FDW]]** 接口——它让 planner 与 executor 能够像访问本地表一样访问外部数据源；Limitless 实现了一个**自定义 FDW** 供 router 访问 shard 数据
- 支持的**索引类型与 PostgreSQL 对齐**
- ⚠️ 注意前述**两层分区**：router 层分区对应 shard，shard 层分区对应 table slice

> **设计红利**：用 PostgreSQL 本就理解的机制（分区表 + 外部表）来表示 Limitless 的表，使系统能够**直接复用 PostgreSQL 优化器**做查询优化。

### 多分片查询的执行

router 生成的计划**尽量把计算推到靠近数据的 shard 侧**：

- 利用「分片表 = PostgreSQL 分区表」这一表示，PostgreSQL planner **天然会为各分区创建子计划**
- router 把这些子计划发给对应 shard，shard **并行执行**
- 最后 router 收集中间结果，必要时做**后处理**（排序、聚合等）
- 只要可能，优化器就把**部分聚合 (partial aggregation) 与排序下推**到 shard，以减少回传 router 的数据量

**示例**：对 Figure 1 中的分片表 `orders` 执行

```sql
SELECT count(*) FROM orders;
```

router 上的 planner 生成如下结构的计划：

```text
Finalize Aggregate
  Output: count(*)
  -> Append
       -> Async Foreign Scan
            Output: (PARTIAL count(*))
       -> Async Foreign Scan
            Output: (PARTIAL count(*))
       ...
```

即通过**异步外部扫描 (async foreign scan)** 在各 shard 上并行求 `count(*)` 的局部值，`Append` 汇总后再求总和。

### 谓词下推 (Predicate Pushdown)

router 把与表关联的谓词分为两类：

| 类别 | 定义 | 处理 |
|------|------|------|
| **Foreign 谓词** | **可以安全地在 shard 上求值**的谓词，例如只涉及内建操作与 `IMMUTABLE` 函数的谓词 | **下推到 shard 求值**，router 再汇总结果 |
| **Local 谓词** | 在 shard 上求值**不安全**的谓词，包括含可变函数（`STABLE`、`VOLATILE`）的谓词，以及需要用**不同于当前用户**的身份执行的 definer 函数 | 把所需数据**拉到 router** 上再求值 |

**示例**：

```sql
SELECT name FROM customers WHERE email = 'abc@xyz.com';
```

router 判定 `WHERE` 子句中的谓词为 foreign，因此把它的求值下推到各 shard，然后汇总结果。

### Join 下推 (Join Pushdown)

Join 可以发生在任意表类型组合之间。

**分片表 × 分片表**：重度依赖 PostgreSQL planner 生成利用既有 **partition-aware join** 算法的优化计划。这对**共置表之间的 inner join** 尤其有效——此时**部分 join 可以在每个 shard 上并行执行**，结果在 router 上汇总。

> ⚠️ **对 PostgreSQL 代价模型的修改**：PostgreSQL 的 cost model **并不知道数据实际驻留在 shard 上**，因此未必总会选中这条优化路径；作者**相应地修改了代价模型**。

示例（共置的 `customers` × `orders`）：

```sql
SELECT c.cust_id, c.name, c.email, SUM(o.amount) AS total_spent
FROM customers c
INNER JOIN orders o ON c.cust_id = o.cust_id
GROUP BY c.cust_id, c.name, c.email;
```

Limitless 会在各 shard 上并行求 join（作为 PostgreSQL foreign scan），然后在 router 上 append 结果。

**分片表 × 参考表**：同样优化。示例：

```sql
SELECT o.order_id, o.amount, t.country, t.tax_rate
FROM orders o
INNER JOIN tax_rates t ON o.tax_rate_id = t.tax_rate_id;
```

由于 `tax_rates` 在所有 shard 上都有副本，系统同样在各 shard 并行求 join 后 append。

> ⚠️ **该优化仅适用于特定 join 类型**：
> - ✅ 适用：笛卡尔积、inner join（无论是否在 shard key 上）、**参考表为 null-padded 一侧**的 outer join
> - ❌ 不适用：**分片表为 null-padded 一侧**的 outer join、以及 anti-join

**其它组合**：

- **standard × standard** → 下推到存放它们的那个特定 shard
- **standard × 其它类型** → **目前只在 router 上执行**

### 函数分发 (Function Distribution)

Limitless 允许**分发（distribute）那些调用只访问单个 shard 数据的 SQL 函数**——这类函数的执行被下推到对应 shard，结果回传 router，从而提升效率。

启用方式：声明待分发函数时，需指定**对应的分片表**以及**哪些函数参数对应 shard key**。示例（基于 Figure 1 的 schema，把函数分发到分片表 `customers`，其第一个参数 `id` 对应 shard key `cust_id`）：

```sql
CREATE FUNCTION func(id INTEGER, n TEXT) RETURNS TEXT
LANGUAGE SQL
VOLATILE
AS $$
    UPDATE customers SET name = n WHERE cust_id = id RETURNING name;
$$;

SELECT rds_aurora.limitless_distribute_function(
    'func(integer, text)', ARRAY['id'], 'customers');
```

### 单分片查询优化：系统的「甜点」

router 执行查询时利用 PostgreSQL 的 **[[Partition Pruning|分区裁剪]]** 机制判断所需源 tuple 是否**全部位于同一个 shard**：

- 若是，则该 **single-shard 查询**被整体下推到该 shard
- 收益：**router↔shard 往返次数更少、延迟更低**

> 🎯 **这使单分片查询成为系统的 sweet spot**。选对 shard key、把函数恰当地标记为 distributed，可以**提高被当作单分片执行的查询比例**，从而改善整体性能。

---

## 实验评估

### 实验设置

| 项目 | 配置 |
|------|------|
| 部署区域 | AWS **us-east-1** |
| 节点类型 | router 与 shard 均为 **Serverless V2** 实例，单节点最高可伸缩到 **256 ACU**（≈512 GB 内存及对应 CPU 等资源） |
| 基准 | 定制版 **[[HammerDB]]**（[[TPC-C]] 衍生的开源基准），修改了建表脚本以支持分片表 |
| 分片表 | `customer`、`stock`、`history`、`warehouse`、`order_line`、`new_order`、`orders`、`district`（均按各自的 `id` 列分片） |
| 参考表 | `item` |
| 查询优化 | 把可在单 shard 内完成的函数标记为 **distributed**，使其直接下推到相关 shard |
| 仓库数 | **12000** warehouses |
| 分布式事务占比 | 约 **10%** |
| 并发 | **1000** 个虚拟用户（并发客户端） |
| 总迭代次数 | **1000 万次** |
| 每配置时长 | **1 小时**，另有 **20 分钟 ramp-up** 阶段（不计入统计） |

### Table 2: 五种测试配置

| Configuration Name | Routers | Shards | Max ACU |
|--------------------|---------|--------|---------|
| r1 | 2 | 4  | 1536 |
| r2 | 4 | 8  | 1536 |
| r3 | 8 | 16 | 1536 |
| r4 | 4 | 8  | 3072 |
| r5 | 8 | 16 | 3072 |

**说明**：r1→r2→r3 是**固定 1536 ACU 下的水平扩展**；r4、r5 则把容量翻倍到 3072 ACU，从而可与 r2、r3 分别构成**垂直扩展**对照。

![[AuroraLimitless_table2_configs.png]]

### Figure 4: HammerDB 吞吐量（New Orders per Minute, NOPM）

![[AuroraLimitless_fig4_nopm.png]]

**说明**：五种配置的 NOPM 柱状图。图中标注值为：

| 配置 | NOPM（图中标注） |
|------|------------------|
| r1 (2×4, 1536) | **1,268,358** |
| r2 (4×8, 1536) | **2,012,763** |
| r3 (8×16, 1536) | **2,042,201** |
| r4 (4×8, 3072) | **2,187,539** |
| r5 (8×16, 3072) | **2,891,718** |

吞吐量随节点数与容量单调递增，最高突破 **289 万 NOPM**，验证了论文摘要中「达到每秒数百万事务」的说法。

> ⚠️ **论文内部数字不一致（需注意）**：
> - 正文 §8.1 称 r1 为 1,268,3**50**，图中标注为 1,268,3**58**
> - 正文 §8.1 称 r4 为 **2,107,539**（据此算出 r2→r4 提升 4.7%），而图中标注 r4 为 **2,187,539**（据此应为 8.7%）
> - 正文 §8.2 在比较「r4 vs r5」时引用的起点值 **2,042,201** 实际是 **r3** 的数值（41.6% 的增幅其实对应 r3→r5）；但该段落引用的延迟起点 13.13ms 确实是 r4 的值。**这段对照的配置标号存在混淆**

### Figure 5: HammerDB 中 NEWORD 存储过程的平均延迟

![[AuroraLimitless_fig5_latency.png]]

**说明**：平均延迟随规模与容量单调下降：

| 配置 | 平均延迟 |
|------|----------|
| r1 (2×4, 1536) | **29.70 ms** |
| r2 (4×8, 1536) | **21.06 ms** |
| r3 (8×16, 1536) | **16.42 ms** |
| r4 (4×8, 3072) | **13.13 ms** |
| r5 (8×16, 3072) | **9.72 ms** |

最优配置 r5 的平均延迟压到 **<10 ms**，对应摘要中「毫秒级延迟」。

### 垂直伸缩效果

**观察**：对于**未饱和**的系统，在保持 router/shard 数量不变的前提下提高最大 ACU 配额可改善性能。

| 对照 | 配置变化 | 吞吐变化 | 延迟变化 |
|------|----------|----------|----------|
| **r3 → r5**（8 routers, 16 shards） | 1,536 → 3,072 ACU | **+41.6%**（2,042,201 → 2,891,718 NOPM） | **−40.8%**（16.42 ms → 9.72 ms） |
| **r2 → r4**（4 routers, 8 shards） | 1,536 → 3,072 ACU | 仅 **+4.7%**（2,012,763 → 2,107,539 NOPM，正文数据） | **−37.7%**（21.06 ms → 13.13 ms） |

**关键洞察**：在 4×8 这个较小规模上，翻倍容量对**吞吐几乎无益（+4.7%）但对延迟改善显著（−37.7%）**——说明此时瓶颈已不在总容量，而在节点数量本身（每个节点的负载已到饱和边缘，多给容量只能缩短单个请求的排队时间而无法提升整体并行度）。这正是需要转向水平伸缩的信号。

### 水平伸缩效果

**观察**：当垂直伸缩因系统饱和而触顶时，水平伸缩通过把负载分散到更多 shard 来改善性能。

| 对照 | 配置变化（容量不变） | 吞吐变化 | 延迟变化 |
|------|----------------------|----------|----------|
| **r1 → r2**（1,536 ACU 不变） | 2 routers/4 shards → 4 routers/8 shards | **+58.7%**（1,268,350 → 2,012,763 NOPM） | **−29.1%**（29.70 ms → 21.06 ms） |
| **r4 → r5**（3,072 ACU 不变） | 4 routers/8 shards → 8 routers/16 shards | **+41.6%**（正文引用 2,042,201 → 2,891,718，⚠️ 起点值实为 r3） | **−26%**（13.13 ms → 9.72 ms） |

### Figure 6: HammerDB 中的 ACU 分配（r1 vs r2）

![[AuroraLimitless_fig6_acu_r1_r2.png]]

**说明**：每条曲线代表**一个 shard 的 ACU 分配**随时间的变化。

- **(a) r1（4 shards）**：峰值期（约 19:45–20:50）出现**显著的不均衡**——4 条曲线的平台值分别约为 **205 / 197 / 188 / 146 ACU**，最高与最低相差近 60 ACU。论文概述为「有的 shard 达到 200 ACU 而另一些只在 150 ACU」。这说明**性能被负载最重的那个 shard 卡住**，而**总 ACU 分配却远低于上限**（4 个 shard 合计约 736 « 1536）——买了容量却用不上
- **(b) r2（8 shards）**：8 条曲线在峰值期（约 11:20–12:30）几乎重叠于 **125～140 ACU** 的**窄带内**（仅一条约 127 略低），容量利用效率显著提高，因而整体性能更好

**共同模式**：三段式曲线——负载开始时 ACU 分配**攀升**，峰值期**稳定**，负载收尾时**逐步下降**，体现了系统根据负载变化**自动调整资源分配**的能力。

### Figure 7: HammerDB 中的 ACU 分配（r4 vs r5）

![[AuroraLimitless_fig7_acu_r4_r5.png]]

**说明**：与 Figure 6 同类对照，在 3,072 ACU 预算下：

- **(a) r4（8 shards）**：峰值期每 shard 分配落在 **130～160 ACU**
- **(b) r5（16 shards）**：峰值期每 shard 分配落在 **85～100 ACU**，**负载分布更均匀**

**结论**：r5 的负载分布比 r4 更均匀，这解释了它更高的吞吐与更低的延迟。整体上，Aurora Limitless 提供了灵活的伸缩选项来支撑动态负载——既可通过增加 ACU 做垂直伸缩，也可通过增加 router 与 shard 做水平伸缩。

> 💡 **两张 ACU 图的共同论证逻辑**：水平伸缩之所以有效，不是因为「总容量变多了」（对照组的 max ACU 是固定的），而是因为**分片数增多让热点被摊平**，从而让系统能真正把预算内的容量用起来。这是本文实验部分最有价值的洞察。

---

## 相关工作对比

### 分布式 PostgreSQL 系统

| 系统 | 隔离级别与一致性 | 伸缩能力 | 与 Aurora Limitless 的差异 |
|------|------------------|----------|---------------------------|
| **[[Citus]]**（现属 Microsoft Azure） | 2PC 保证分布式事务的 A/C/D，但**不提供 SI** → 暴露大量事务异常 | 支持分片 | Limitless 提供完整 SI 语义 |
| **[[Greenplum]]** | 用**中心化 coordinator** 分配分布式事务 ID 来提供 SI | OLTP + OLAP 混合 | coordinator 在高负载下易成瓶颈；且需维护 local↔distributed 事务映射做可见性检查，带来性能开销与实现复杂度。Limitless 用**时间戳并发控制 + 2PC** 彻底去掉中心协调者 |

Limitless 还额外同时提供**垂直与水平**两种伸缩机制——这个组合在上述系统中都不存在。

### 地理分布式 SQL 数据库

| 系统 | 时间戳机制 | 隔离级别 | 局限 |
|------|-----------|----------|------|
| **[[Spanner]]** | 硬件同步时钟 + **[[TrueTime]]** API，开创了此路线 | 严格可串行化 | 面向跨 region |
| **[[Clock-SI]]** | **松散同步时钟**，无中心时间戳权威 | SI | 学术方案；读需**延迟等待**本地时钟推进（Limitless 用 HLC 消除了这个等待） |
| **[[CockroachDB]]** | **[[Hybrid Logical Clock]] (HLC)** | 支持 serializability 与 RC，**不支持 SI** | 一般**不保证外部一致性** |
| **[[YugabyteDB]]** | HLC | 支持 serializability 与 SI | HLC 有时需**等出集群最大时钟偏移**；缺少 TrueTime/Time Sync 时该值须保守设到**数百毫秒** |
| **[[Aurora DSQL]]** | — | **乐观并发控制**提供 SI | serverless，支持**多 region 读写** |
| **[[Azure Cosmos DB]]** / **[[MongoDB Atlas]]** | — | — | 提供多模型支持，但**常以一致性换可用性** |
| **[[PolarDB]]** | **RDMA-based timestamp oracle** 或 HLC | — | HLC 路线以**放松一致性**换取更低延迟 |

**Aurora Limitless 的定位差异**：专为**跨 AZ 而非跨 region** 场景设计。这个收窄的作用域让它能针对跨 AZ 需求做优化，并**直接复用 Aurora 已有的跨 AZ 存储层**来保证持久性，从而**完全不需要引入 Paxos 或 Raft 这类额外共识协议**。它通过与 [[Amazon Time Sync]] 的深度集成获得强一致性，同时保持高可用——设计上同时吸收了 [[Spanner]] 与 [[Clock-SI]] 的洞察。

### 伸缩与架构

| 类别 | 代表系统 | 特点 |
|------|----------|------|
| 传统云数据库 | **[[Azure Database for PostgreSQL]]**、**[[Google Cloud SQL]]** | 聚焦**单实例内的垂直伸缩** |
| 分片中间件 | **[[Vitess]]**、**[[Apache ShardingSphere]]** | 提供水平伸缩，但**运维管理复杂** |
| 存算分离 | **[[Amazon Aurora]]**（本文基础）、**[[Snowflake]]** | Limitless 扩展了 Aurora 的存算分离，路线类似 Snowflake 但**面向 OLTP 优化** |
| 分布式 SQL 处理 | **[[F1]]** | 已证明大规模分布式 SQL 处理可行；Limitless 的贡献在于**最小化跨分片通信** |
| 经典分布式查询 | [[MapReduce]]、[[Dynamo]]、[[TiDB]]、[[OceanBase]]、[[Snowflake]] | Limitless 的查询处理借鉴这些经典研究，但**重点优化单分片查询** |
| 事务协议 | 经典 **2PC** | Limitless 做了两项修改：**容忍 router 故障**、**支持时间戳并发控制** |

---

## 生产经验与未来工作（论文自述）

### 迁移是最大的客户痛点

从单机 PostgreSQL 或自管分片迁移到分布式 PostgreSQL 是客户面临的**最大挑战**。许多既有应用在开发时**完全没有考虑表或 schema 分区**。为了让这类应用可扩展，客户必须：

1. 判断自己的负载**是否可分区**
2. 设计一个 **scale-out ready 的 schema**
3. 把数据**迁移到新 schema**
4. **移植**既有 SQL 查询与代码

> ⚠️ **shard key 的选择是关键考量：它是一道「单向门」（one-way door），改变它需要重新分片并迁移数据。** 作者认为构建帮助应对这些挑战的**工具**是必要的。

### PostgreSQL 需要现代化的地方

PostgreSQL 是构建分布式数据库「极其灵活的基座」，但作者在实现中「绊倒在许多可以受益于现代化改造与分布式适配的 PostgreSQL 行为上」：

1. **Schema 演进**：高可用的分布式应用需要 [[Schema Versioning|schema 版本化]]（如 [[F1]] 的在线异步 schema 变更）、canary 发布，以及**针对 schema 的 MVCC**（如 Azure SQL 的 MD-MVCC）
2. **提交序 ≠ 可见序**：分布式事务在「**事务提交的顺序与它们对其它事务变得可见的顺序一致**」的系统中要容易实现得多。但**社区版 PostgreSQL 并非如此**——这已经在带只读副本的集群中导致了异常。Limitless 通过**用时间戳同时定义两者**（Property 1）强行让提交序与可见序对齐
3. **不支持可串行化的理由**：作者**主动放弃了 serializability**，原因有两条：① **客户需求有限**；② 在 PostgreSQL 的 SSI 实现中，**事务的序列化顺序与其提交顺序并不一致**——这与上一条的设计哲学冲突
4. **混合伸缩需精细调优**：作者发现 hybrid scaling 需要**仔细调参**才能为客户最大化性能与成本收益

---

## 批判性思考

### 优点

1. **工程与理论的高质量结合**：Property 1 是一条极简的规范，而后续 §5.2–§5.6 的所有机制（时间戳可见性、HLC 推进、lead shard 询问、COMMITTING 等待、commit wait、租约）都可以被理解为「为在各种并发/故障场景下守住 Property 1 而打的补丁」。论文对 commit wait 的三步实时序证明写得很干净。
2. **Lead Shard 的设计非常务实**：这是全文我最欣赏的一点。它没有为了「协调者高可用」而给 router 加 standby（成本高）或引入 Raft（复杂度高），而是**把状态搬到本来就有 standby 的组件上**——用一个「谁有备机就把状态放谁那里」的朴素观察，把可用性问题转化成了零成本的责任重分配。
3. **HLC 消除读延迟**：相比 [[Clock-SI]] 的「延迟读直到本地时钟追上」，用 $C \leftarrow \max\{C, \text{startTs}+1\}$ 把等待成本从「读路径」转移到「后续写路径的时间戳抬升」上。这是典型的**把开销移到不敏感路径**的优化，对读多写少的 OLTP 很划算。
4. **对 PostgreSQL 已有机制的极致复用**：分片表 = 分区表 + FDW 外部表，slice = 分区表的分区，commit 时间戳映射塞进 commit log，`DETACH PARTITION` 复用于 shard split。这种「不发明新抽象、只借壳」的做法极大降低了实现与维护成本，也是它能保持高 PostgreSQL 兼容性的根本原因。
5. **shard split 卸载到存储层**：用 [[Copy-on-Write Cloning]] 让「已经过载的 shard 也能被分裂」——直击了自动分片系统最尴尬的死锁场景（越需要分裂的节点越没有余力做分裂）。
6. **实验的论证逻辑清晰**：Figure 6/7 的 ACU 曲线不是简单的性能展示，而是**用「热点摊平」解释了水平伸缩为什么在固定容量预算下仍能提升 41–59% 吞吐**——即瓶颈是最热 shard 而非总容量。这个洞察比单纯的 NOPM 柱状图有价值得多。
7. **生产验证充分**：已生产运行 1 年以上，最大 64 shards + 32 routers，作者也坦诚地写出了客户迁移的痛点与 PostgreSQL 的不足。

### 局限性

1. **实验规模远小于生产规模，且缺少横向对比**
   - 实验最大只到 **16 shards**，而生产已有 **64 shards**——最有说服力的大规模数据反而没有展示
   - **完全没有与 [[Citus]]、[[Greenplum]]、[[CockroachDB]]、[[YugabyteDB]] 等竞品的性能对比**，Related Work 里的比较全是定性的机制描述。因此「近线性可扩展」的说法缺少同类基准的参照
   - 只有 HammerDB 一个基准（且是定制版），没有 sysbench、YCSB 或真实生产负载的 trace 回放

2. **论文数据存在自相矛盾**（如 Figure 4 说明中标注）：r4 的 NOPM 在正文（2,107,539）与图中（2,187,539）不一致；§8.2 比较「r4 vs r5」时引用的吞吐起点值实际是 r3 的。这类错误在 SIGMOD 论文中不应出现，也让读者难以判断 +4.7% 这个关键结论的可信度。

3. **关键机制缺少量化评估**——这是我认为最大的遗憾。论文详述了七套精巧机制，但**没有任何一个给出开销数据**：
   - **2PC 的开销**有多大？分布式事务占比从 10% 升到 50%、90% 时性能如何衰减？（10% 这个比例明显是被精心挑选的友好设置）
   - **commit wait** 究竟给延迟加了多少？作者只说「rarely adds latency」，但没有 CEB 与存储延迟的分布对比数据
   - **lead shard 询问**（难题 2）在写热点下会触发多少跨 shard RPC？
   - **COMMITTING 等待**（难题 3）的实测阻塞时长？
   - **shard split 期间** switchover 窗口有多长？多少事务被终止？作者说「唯一影响是 switchover」，但完全没给这个窗口的量化数据——而这恰恰是客户最关心的运维风险
   - **HLC 免等待**相比 Clock-SI 的延迟做法到底省了多少？这是核心创新之一，却没有 A/B 对照

4. **对时钟基础设施的强依赖**：整个正确性论证建立在 [[Amazon Time Sync]] 的 CEB **永不被违反**这一假设上。论文完全没有讨论：
   - 若 Time Sync 出现**故障或 CEB 显著膨胀**（例如卫星链路问题），系统的行为与降级策略是什么？
   - 若真实时钟**实际偏出了声称的界限**（即 CEB 假设被破坏），会产生什么一致性异常？
   - 这也意味着该设计**基本无法移植**到没有等效时钟服务的环境（这一点比 Spanner 更甚，因为 Spanner 至少讨论了 TrueTime 失效时的行为）

5. **放弃可串行化的理由不够充分**：以「客户需求有限」为主要理由略显单薄。虽然作者补充了「PostgreSQL SSI 的序列化顺序与提交顺序不一致」这个技术理由，但对于金融类场景（论文 Introduction 自己举的例子），SI 的 write skew 异常是真实风险，值得更严肃的讨论。

6. **跨类型 join 的能力缺口**：**standard 表与其它类型表的 join 只能在 router 上执行**（完全不下推），这在数据导入后的过渡期可能是显著的性能陷阱。此外 outer join/anti-join 在分片表为 null-padded 侧时无法下推，这些限制的实际影响没有评估。

7. **死锁检测策略偏简单**：分布式死锁靠**客户配置超时**触发，且**随机选受害者**中止。既无自适应超时，也无基于代价（如已完成工作量、事务优先级）的受害者选择策略，在高争用负载下可能造成大量无谓回滚。

8. **水平伸缩的自动化程度有限**：虽然支持自动 shard split，但**没有 shard merge**（缩容），也**不支持 shard key 的在线变更**（作者自己称之为「单向门」）。这意味着水平伸缩实际上是**单调递增**的，对于有明显周期性峰谷的负载，只能靠垂直伸缩回收成本。

### 潜在改进方向

1. **补齐机制级微基准**：针对分布式事务比例、shard split switchover 窗口、commit wait、lead shard 询问频率分别做 ablation，这会让论文的技术贡献可信度大幅提升。
2. **时钟降级路径**：设计 CEB 膨胀或 Time Sync 不可用时的降级模式（例如退回 Clock-SI 式的延迟读、或主动限制提交速率），并给出一致性/可用性的权衡曲线。
3. **Shard merge 与在线 reshard**：支持缩容与 shard key 在线变更，把「单向门」变成可逆操作。前者可复用 slice 迁移机制，后者可能需要借鉴 [[F1]] 的在线 schema 变更思路。
4. **自动 shard key 推荐工具**：正如作者在未来工作中所述，可以从生产负载 trace 中分析访问模式，自动推荐分片键与共置方案，并预估单分片查询占比。
5. **跨类型 join 的下推**：至少把 standard 表当作一个特殊的「单分片 reference 表」来处理，使 standard × sharded 的 join 能部分下推。
6. **更智能的死锁处理**：自适应超时 + 基于已完成工作量的受害者选择；或考虑用 wait-die / wound-wait 这类基于时间戳的预防策略（系统本来就有全局可比的时间戳，这是天然优势）。
7. **Serializability 的可选支持**：在 SI 之上实现 SSI，且利用已有的全局时间戳让序列化顺序与提交顺序对齐——这恰好能解决作者所抱怨的 PostgreSQL SSI 的问题。

### 可复现性评估

- [ ] 代码开源（**闭源商业服务**，仅可通过 AWS 付费使用）
- [ ] 预训练模型（不适用）
- [x] 实验配置描述完整（Table 2 给出五种配置，warehouse 数、并发数、迭代数、时长齐全）
- [x] 基准可获取（[[HammerDB]] 开源，但**论文的定制修改未公开**）
- [ ] 关键机制的量化数据（如上所述，7 套机制全部缺少开销数据）

> **总评**：作为一篇 **Industry Track** 论文，它在**系统设计的讲述**上是优秀的——lead shard、HLC 免等待、存储层克隆分裂、租约防陈旧读这几个设计都值得借鉴。但作为**学术评估**，它的实验部分明显偏弱：既无竞品对比，也无机制级 ablation，还存在数据前后矛盾。**读它应当为了学设计思路，而非为了信性能数字。**

---

## 关联笔记

### 基于

- [[Amazon Aurora]]：存算分离架构、3-AZ 分布式存储、写时复制克隆的直接来源
- [[Aurora Serverless V2]]：垂直伸缩的 ACU 调容基础设施与非对称伸缩速率策略
- [[Amazon Time Sync]]：有界误差时钟服务，全部时间戳与一致性保证的根基
- [[Clock-SI]]：用松散同步时钟实现 SI 的思路来源（Limitless 用 HLC 改进了其读延迟）
- [[Spanner]]：commit wait 与硬件时钟驱动事务排序的先驱
- [[Two-Phase Commit]]：经典 2PC 协议，Limitless 做了 lead shard 与时间戳两项修改
- [[Hybrid Logical Clock]]：消除跨节点时钟偏移读延迟的手段
- [[PostgreSQL]]：整个系统的基座（分区表、FDW、commit log、行版本格式、heavyweight lock 都被复用）

### 对比

- [[Citus]]：同为分布式 PostgreSQL，但不为分布式事务提供 SI
- [[Greenplum]]：同为分布式 PostgreSQL 且提供 SI，但依赖中心化 coordinator
- [[CockroachDB]]：HLC 路线，支持 serializability 但无 SI，且不保证外部一致性
- [[YugabyteDB]]：HLC 路线，支持 SI，但无 Time Sync 时须等出数百毫秒时钟偏移
- [[Aurora DSQL]]：同门产品，走乐观并发控制 + 多 region 路线
- [[PolarDB]]：RDMA timestamp oracle 或 HLC，后者放松一致性换低延迟
- [[Vitess]] / [[Apache ShardingSphere]]：分片中间件路线，运维复杂
- [[Snowflake]]：同为存算分离，但面向 OLAP
- [[F1]]：大规模分布式 SQL 处理与在线 schema 变更的参照
- [[TiDB]] / [[OceanBase]]：其它大规模分布式 OLTP/HTAP 系统

### 方法相关

- [[Snapshot Isolation]]：Repeatable Read 对应的核心隔离级别
- [[Read Committed]]：另一个支持的隔离级别（逐语句取快照 + 关闭写冲突检测）
- [[Repeatable Read]]：PostgreSQL 语义下等价于 SI
- [[MVCC]]：多版本并发控制的基础
- [[Timestamp-Based MVCC]]：用标量时间戳替代 xid 集合快照的核心创新
- [[External Consistency]]：比实时序 SI 更强的保证
- [[Commit Wait]]：保持实时序的关键等待
- [[Lead Shard]]：2PC 状态持久化位置的关键设计
- [[Table Slice]]：shard split 的最小迁移单元
- [[Copy-on-Write Cloning]]：存储层卸载分裂开销
- [[Lease]]：防止 zombie shard 提供陈旧读
- [[Foreign Data Wrapper]]：router 访问 shard 数据的接口
- [[Partition Pruning]]：识别单分片查询的机制
- [[Predicate Pushdown]] / [[Join Pushdown]]：计算下推优化
- [[Aurora Capacity Unit]]：伸缩的计量单位
- [[Proportional ACU Allocation]]：按消耗比例分配容量配额
- [[Sharding]]：水平分片的基本范式
- [[Data Co-location]]：共置以保留 join 本地性
- [[Distributed Deadlock Detection]]：waits-for 图并集检测环
- [[Global Sequence]]：按号段分发的全局序列
- [[Split-Brain]]：靠存储层单写者语义规避
- [[Serializability]]：主动未支持的隔离级别

### 数据/基准相关

- [[HammerDB]]：本文唯一使用的基准（TPC-C 衍生）
- [[TPC-C]]：HammerDB 的原型基准
- [[ClockBound]]：Time Sync API 的开源 daemon

---

## 速查卡片

> [!summary] Aurora PostgreSQL Limitless Database (SIGMOD Companion '26)
> - **核心**：用 [[Amazon Time Sync]] 的有界误差时钟把标量时间戳做成快照与提交序的唯一定义，从而在无中心事务管理器的前提下把 PostgreSQL 完整语义扩展到分片集群
> - **架构**：Router 层（无数据、持元数据、驱动查询与提交）+ Shard 层（持数据分片）+ Aurora 3-AZ 分布式存储 + 控制平面
> - **三种表**：sharded（哈希分片，可共置）/ reference（全 shard 复制）/ standard（单 shard）
> - **并发控制**：时间戳 MVCC（`xmin.commitTs ≤ startTs < xmax.commitTs`）+ 行锁写冲突检测；RR(=SI) 与 RC 两级隔离
> - **提交协议**：2PC，状态持久化在有 standby 的 **lead shard**（router 无 standby），commitTs = max(各 prepareTs)；只读与单分片写事务**绕过 2PC**
> - **一致性三招**：HLC 抬时钟免读延迟 / 向 lead shard 询问 prepare 中事务 / 等待 COMMITTING 事务定论
> - **外部一致性**：commit wait 等到 `now().earliest > commitTs`，与写存储并行执行故几乎不上关键路径
> - **伸缩**：垂直（Serverless V2 按 ACU 比例分配配额，scale-up 快 / scale-down 慢）+ 水平（512 slices/表，写时复制克隆 → redo 回放 → switchover → 清理）
> - **查询**：分片表映射为「分区表 + FDW 外部表」，复用 PostgreSQL planner 做部分聚合/排序/谓词/join 下推；**单分片查询是甜点**
> - **结果**：HammerDB 12000 仓 / 1000 并发，最高 **2,891,718 NOPM**、平均延迟 **9.72ms**；固定 1536 ACU 下 2×4→4×8 吞吐 **+58.7%**（靠摊平热点而非加容量）
> - **生产**：AWS 生产运行 1 年+，最大 64 shards + 32 routers，最常用 4 routers + 8 shards
> - **主要短板**：无竞品对比、无机制级 ablation、分布式事务仅占 10%、正文与图表数据有矛盾
> - **代码**：闭源商业服务（[Aurora Limitless](https://aws.amazon.com/rds/aurora/)）

---

*笔记创建时间: 2026-07-31*
