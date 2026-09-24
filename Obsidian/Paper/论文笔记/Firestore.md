---
title: "The Live Database: Firestore's Scalable and Consistent Realtime Queries"
method_name: "Firestore"
authors: [Matt Bell, David Gay, Per Jacobsson, Ram Kesavan, Namita Lal, Xijiao Li, Hui Wu]
year: 2026
venue: PVLDB 19(12)
tags: [realtime-query, serverless-database, nosql, query-matching, strong-consistency, change-stream, document-database, spanner]
zotero_collection: VLDB/2026
image_source: local
created: 2026-09-24
---

# 论文笔记：The Live Database: Firestore's Scalable and Consistent Realtime Queries

## 元信息

| 项目 | 内容 |
|---|---|
| 机构 | Google（含 University of Illinois Urbana-Champaign） |
| 日期 | 2026，PVLDB 19(12): 4130–4142 |
| 项目主页 | — |
| 对比基线 | [[InvaliDB]]、[[RethinkDB]]、[[MongoDB]] Realm / Couchbase Lite |
| 链接 | [DOI](https://doi.org/10.14778/3827998.3828021) |
| 原始 PDF | `VLDB/2026/Google Firstore (DocDB).pdf` |

---

## 一句话总结

> Firestore 用 [[ChangeLog]] + [[QueryMatcher]] + [[Frontend]] 三层实时栈，把 Spanner 的提交流改造为「变更流 + 时间推进消息」，从而在无持久化 changelog 的前提下，为千万级长连接提供强一致、低延迟、可自动扩缩的[[Realtime Query|实时查询]]。

---

## 核心贡献

1. **首个生产级实时查询内幕披露**：作者称这是第一篇完整呈现生产数据库系统如何同时做到自动扩缩、高可用、强一致与低延迟实时查询的论文（相对其 2023 ICDE 的 Firestore 总览论文 [[Firestore|ICDE 2023]] 做了深挖）。
2. **无持久化变更日志的一致性方案**：[[ChangeLog]] 与 Backend 之间用两阶段 Prepare/Accept 协议（**不是** 传统 [[Two-Phase Commit]]），把「提交时间上下界 + before/after 文档快照 + [[Safetime]] 心跳」组合成可重放的变更流，避免了在 Spanner 上维护新表带来的热点与成本。
3. **可扩展的查询匹配索引**：[[ChangeStream Filter]] 用「字段列表 → [[Interval Tree]]」的 [[Trie]] 结构，把单次变更的匹配代价从 $O(\text{queries})$ 降到 $O(\text{matches}) + O(\log \text{queries})$，并在 pre-image / post-image 上取差集得到「新命中」与「不再命中」的查询。
4. **跨查询的强一致快照协议**：Frontend 维护按提交序的缓冲区，依靠各分片 [[Safetime]] 的最小公共提交版本决定何时可以释放文档变更，并周期性下发 [[Global Snapshot]]，使同一连接上多条实时查询永远看到同一时间戳的视图。
5. **SDK 侧的离线与乐观推测**：客户端维护两区缓存（已确认区 + 待提交区），本地写先推测性地反映到活跃查询，再异步提交，从而支持完全离线使用与重连后的自动对账。

---

## 问题背景

### 要解决的问题

社交、聊天、游戏这类移动/Web 应用共享一组需求：自动且实时的数据更新、海量并发用户、自动透明故障切换、网络中断时的平滑降级与重连对账。传统做法需要开发者自建应用服务器、分片数据库、流式协议、负载均衡与自动扩缩器。

论文的贯穿例子是一个**全球聊天应用**：

- `/users/{uid}` 账户文档（如 `/users/owl`）
- `/rooms/{rid}` 聊天室文档（如 `/rooms/C`）
- `/rooms/{rid}/messages/{mid}` 消息文档（如 `/rooms/C/messages/22`）

对应三条主要实时查询：

```sql
select * from /rooms order by name
select * from /rooms/C/messages where time > T order by time desc
select * from /**/messages where id in mentions and time > T order by time desc
```

第三条里的 `/**/` 是 [[Collection Group Query|集合组查询]]，可跨同名的所有集合读取。

### 现有方法的局限

1. **读写放大不匹配**：其他实时数据库依赖把整库变更流推到单节点，写扩展性受限（RethinkDB、Meteor、Parse）。
2. **一致性弱**：[[InvaliDB]] 最接近 Firestore（类似的 filter/布尔/排序能力、按写分片 + 查询副本），但查询是[[Eventual Consistency|最终一致]]且分片看起来是手动的；MongoDB Realm 与 Couchbase Lite 也不提供强一致快照，且要求本地复制整库。
3. **Spanner 原生 changelog 不适用**：理论上可用 Spanner Change Streams 建一张新表实现变更日志，但实时栈的访问模式会让它容易**热点化且成本高昂**。
4. **多租户的吵闹邻居**：共享任务处理多客户数据库，单库突发流量会拖垮其他库的延迟与错误率。

### 本文的动机

如果把「提交序号」这件事从存储层解耦出来，交由一个只需保证「完整或明确告知不完整」的旁路组件（ChangeLog）来推进，那么实时查询就可以建立在 Spanner 已有的强一致提交之上，同时避开追加型日志表的热点问题；再配合面向写负载的独立分片（[[Slicer]]）与面向连接/查询的横向扩展，读写两侧就能各自独立伸缩。

> ⚠️ 需要说明：本文**没有任何编号公式**，正文也几乎不使用数学记号——它的「形式化」体现在协议步骤、时间轴表格与数据结构定义上。下文「关键公式」一节记录的是论文中定义的核心形式化对象与判定条件，而非原文公式的抄录。

---

## 方法详解

### 系统架构

Firestore 的实时栈建立在 [[Spanner]]（持久化与事务）、[[TrueTime]]（时间上下界）、[[Chubby]]（选主）、[[Slicer]]（自动分片）之上。整体分为四个可各自扩到数千任务的组件（Figure 1）：

- **Frontend**：对外 API 端点，管理连接与查询生命周期
- **Backend**：执行读写事务，与 ChangeLog 走两阶段协议
- **Metadata Cache**：分片到任务的映射
- **Realtime Cache**：即实时栈本体（Frontend / [[QueryMatcher]] / [[ChangeLog]]）

实时栈内部按树状组织（Figure 2）：**一个 ChangeLog 喂多个 QueryMatcher，多个 ChangeLog 组共同覆盖整个 keyspace**。文档 key 空间按**写负载**自动切分（与 Spanner tablet 按读写负载切分相互独立），多个分片归属同一 group；读侧则通过 Frontend 横向扩展 + 在同一分片上的多个 QueryMatcher 之间做连接负载均衡来实现。

- **输入**：客户端发起的实时查询 + 数据库写操作
- **核心模块**：[[ChangeLog]]（写路径）、[[QueryMatcher]]（匹配）、[[Frontend]]（定序与下发）、客户端 SDK（缓存与推测）
- **输出**：以提交序下发的文档变更 + [[Global Snapshot]] 消息
- **关键指标**：变更从提交到到达各层的延迟（Table 1）、通知延迟 p50/p90/p99

### 核心模块

#### 模块1: 查询生命周期与 Frontend 定序

**设计动机**：QueryMatcher 按**到达序**而非**提交序**处理变更，且各分片推进速度不同，因此必须有一层负责把乱序输入重排成提交序输出。

**具体实现**：

- 新查询先以 **polling mode**（轮询模式）发往 Backend 取初值，完成后切到 **streaming mode**（流式模式）
- Frontend 计算查询的 keyspace 范围（单文档 / 单集合 / 集合组），查 Metadata Cache 找到覆盖该范围的分片及其 QueryMatcher，再转发查询
- Frontend 维护**按提交序的缓冲区**：当查询涉及的所有分片的 [[Safetime]] 最小公共提交版本越过某变更的提交版本时，才可安全释放该变更
- 同样的机制被复用于跨查询一致性——向客户端发送 [[Global Snapshot]] 消息，表示该连接上**所有**活跃查询都已推进到某个提交版本
- **limit / offset 查询**需额外协调：Frontend 维护结果集大小、键与边界元数据；当某文档离开满额结果集时，发起一次轮询查询补洞
- **不完整流的检测**：若某分片在某时间区间上的变更流不完整（任务失败等），QueryMatcher 会拒绝对应查询，Frontend 让这些查询停留在 polling mode，直到读到超出该不完整区间的读时间

#### 模块2: QueryMatcher 与 ChangeStream Filter

**设计动机**：暴力做法是遍历所有查询逐个比对 filter，代价与 QueryMatcher 上的活跃查询数成正比，不可接受。

**具体实现**：[[ChangeStream Filter]]（Figure 3）是一个内存结构，形态为「**字段列表 → [[Interval Tree]]**」的映射：

```
{ ["mentions"]          : <interval_tree1>,
  ["mentions", "sender"]: <interval_tree2>,
  ["sender"]            : <interval_tree3>,
  ["sender", "time"]    : <interval_tree4>,
  ["text"]              : <interval_tree5>, ... }
```

- 每条区间树节点还挂着「命中该区间的查询 ID 列表」
- 该表示对 Firestore 的 filter 表达能力是**精确且完备**的：字段与常量的比较（等值、`in`）、合取、析取，且**至多一个关系运算符**
- 匹配方式是：用文档的字段值元组在映射的每个条目中查找，然后对 **pre-image 与 post-image** 的结果集取差 —— 前者给出「曾经命中、现已不再命中」，后者给出「当前命中」
- 映射本身用 [[Trie]] 实现以复用前缀（每个节点持单个字段，根到节点的路径即字段列表）
- 区间树是**按区间下界排序的 AVL 树**，每个节点额外记录子树内的最大上界以便高效搜索

#### 模块3: ChangeLog 与两阶段 Prepare/Accept 协议

**设计动机**：Spanner 目前不提供序列化的变更日志；用新表实现会热点化且昂贵。因此改为**不持久化 changelog**，而由「变更 + 时间推进消息」让上游重建出强一致的变更序列。

**具体实现**（Figure 4，成功路径）：

1. Backend 在向 Spanner 发提交前，向 ChangeLog 发 **Prepare**，携带事务中的文档 key
2. ChangeLog 基于 [[TrueTime]] 调用确定可接受的提交时间**上下界**并回复
3. 同时 ChangeLog 向上游发 **blocked shard** 消息，表示该分片范围存在在途事务、[[Safetime]] 推进需被压制在下界（覆盖常规心跳）
4. Backend 用带时间界保证的内部 Spanner API 提交事务
5. Spanner 返回成功或失败
6. 无论成败，Backend 发 **Accept**，携带变更的文档前后快照与**实际提交时间**
7. ChangeLog 向上游转发已提交数据
8. 随后发 **unblocked shard**，撤销对 safetime 的压制

**与 2PC 的关键区别**：Backend 收到 Prepare 响应后即可自由提交，**即使 ChangeLog 在 Prepare 与 Accept 之间失败也不会回滚 Spanner 事务**。实时栈检测到自己持有不完整的消息流后，把受影响的实时查询退回 polling mode 来恢复——即「存储向前、通知降级」。

**跨分片事务**：若一次变更涉及多个分片，Backend 并行与多个 ChangeLog 交互，只有收到**全部** Prepare 成功响应后才提交。

**超时与所有权变更**：ChangeLog 等待 Accept 超时，则该时间区间（由上下界框定）被标记为不完整；ChangeLog 因故障切换或重新分片而获得新分片所有权时，也会发出同样的「不完整」信号。

#### 模块4: 客户端 SDK 的缓存与推测执行

**设计动机**：离线可用性是移动应用的关键。SDK 要让在线/离线对开发者看起来一致。

**具体实现**：

- 本地文档缓存按环境选择 [[LevelDB]]（iOS）/ SQLite（Android）/ IndexedDB（浏览器），存四类内容：服务端返回的文档、本地写入的变更、加速查询的本地索引、缓存管理的辅助数据
- 缓存分两区：**zone 1** 是截至最近一次 [[Global Snapshot]] 的服务端文档；**zone 2** 是尚未被 API 服务处理的待提交变更
- 应用首次发起查询时**先对本地缓存执行并立即返回推测结果**（一致但可能不完整的数据 + 用户近期写入），随后才与服务端建立查询
- 本地写先入 zone 2，并在发送前就推测性地反映到所有活跃查询（即时反馈）；变更按**发起顺序**串行发送，后者需等待前者被接受或拒绝
- 写入被服务端拒绝（文档已删除、权限变化等）时，变更被丢弃并回滚缓存与活跃查询；应用可注册异步回调获知结果
- 离线时新查询按 best-effort 处理，缓存中没有相关数据则返回空结果，恢复在线后更新为非推测结果

### 协议一致性（Section 6）

Frontend 与客户端之间的协议是「[[Global Snapshot]] 消息」与「快照之间的变更描述」交替的序列。设相邻两个全局快照时间为 $\mathbb{t}_1$ 与 $\mathbb{t}_2$（原文用双线体记号），则其间消息为：

1. 对每个「在 $\mathbb{t}_2$ 新进入查询结果集」或「自 $\mathbb{t}_1$ 起被修改」的文档，至少一条变更通知；同一文档的多条通知按**提交序**排列
2. 对每个「在 $\mathbb{t}_1$ 属于查询 $Q$ 的结果集、在 $\mathbb{t}_2$ 不属于」的文档，二选一：发**删除通知**，或（在 polling mode 下）附带 $Q$ 在 $\mathbb{t}_2$ 的结果集信息供 SDK 校验——即**结果总数**与**结果文档键的 [[Bloom Filter]]**。SDK 用该 Bloom filter 从自己在 $\mathbb{t}_1$ 的已知结果集中得到「确定已不在结果集」的文档数 $d$：若 `(变更通知带来的新增结果数) − d == 结果总数的变化量`，则删除被精确确定；否则 $Q$ 被作废并透明地重跑

协议**不保证**按 `ORDER BY` 下发通知（流式下不可行），也**不保证**每个查询变更都有对应通知（轮询下不可行）；靠全局快照 + 客户端排序 + 轮询校验来保证用户看到的一致有序视图。

### 可扩展性与可用性（Section 7）

**三个正交的读扩展维度**：

1. 端设备直连：Frontend CPU 主要消耗在活跃连接上（与轮询返回的文档数、增量更新量成正比），按 CPU/内存横向扩展
2. 新查询 / 重轮询带来的 polling 请求：量级由终端用户活动决定，依赖 Backend 与 Spanner 的负载均衡与分裂能力
3. QueryMatcher 扇出：ChangeLog → QueryMatcher → Frontend 的扇出使通知可扩到大量设备；组内 ChangeLog 与 QueryMatcher 的映射目前**静态设为 1:3**（按当前生产需求），第 9 节提到可通过自定义扇出配置应对异常负载

**写扩展**：每个 ChangeLog 任务周期性向 [[Slicer]] 上报负载指标（流量速率、载荷大小），Slicer 决定切分/合并；调整间隔设为**几分钟**量级，避免对短时突发过度反应。

**高可用**：ChangeLog 进入写路径的 Prepare 阶段，因此它的可用性直接决定写可用性。

- ChangeLog 任务组织为跨可用区的**副本组**，分片按副本组而非单任务分配
- 组内用基于 [[Chubby]] 的选主（法定人数 3–5 个 Chubby cell）选出一个 leader 处理该组分片的全部写入；非 leader 不参与事务、也不持有 changelog 状态
- 选主典型耗时：单 region p50 100ms / p99 1.5s；multi-region p50 400ms / p99 2s —— 快到 Backend 常能直接向新 leader 重试两阶段协议并完成写入
- 另配合流控、负载削减，以及在卡线程 / 无法触达 [[Slicer]]、[[Spanner]] 等关键依赖时主动放弃 leadership 或自杀的启发式；活跃探针负责快速检测无响应的 ChangeLog 并触发受控重启
- ChangeLog 与 Chubby 的部署区与 Firestore 的 Spanner 拓扑**显式对齐**，leader 选举尽量复刻 Spanner 的做法，以优化写延迟并让故障域对齐
- **分片切分对查询的影响**：ChangeLog 通知上游 QueryMatcher，后者再告知 Frontend，Frontend 把受影响查询退回 polling mode 并重新路由到新分片的所有者

**多租户与性能隔离**：每个任务服务同 region 内多个客户的数据库。隔离手段是对**每库**施加宽松资源约束以保证公平，并给各组件预置额外余量让自动扩缩有时间反应。Firestore 要求合规流量按「**500/50/5 规则**」渐进增长：从 500 QPS 起，每 5 分钟最多增加 50%。阈值过激进会伤害客户并污染 Autopilot / Slicer 的输入信号，过宽松则削弱隔离保护，实际阈值由测试与生产流量实验确定。

### 工程实践（Section 9）

- **隔离池（isolation pools）**：各组件的并行部署，带专属容量，SRE 可快速、临时地把反复违规的流量移出共享池，为工程团队争取分析时间。早期用「每次写入的文档数」估计资源开销，对超大文档（解析开销高）失效——因为 Firestore 会传播完整的 before/after 文档来评估谓词，大文档上的实时查询会带来显著带宽与内存开销，属于反模式。自动限流机制每天触发多次而无需人工介入，SRE 干预与隔离池**每年仅约 5 次**
- **隔离池迁移的一致性**：把流量从一个 ChangeLog 池挪到另一个时，所有下游消费者必须同步迁移且不能漏掉变更——为此在分片所有权机制中加了专门的 pool change 协议（因篇幅未展开）
- **SLO 重定义**：实时栈不同于常见的请求-响应系统，SLO 围绕**查询与客户端连接的生命周期**而非单次请求定义；并用探针生成模拟真实/异常应用的人工流量（建长连接、建实时查询、发写、校验数据到达）来覆盖整个部署
- **编程模型**：早期用共享内存 + 传统并发原语（锁）导致 mutex 争用、悬垂指针类 bug 频发；改为**基于请求队列的 actor model** 框架后，强制开发者显式表达并发行为，代码「更显然正确」，虽然某些场景要额外工作来避免瓶颈，但净收益为正
- **监控**：用 Monarch 做异常检测与告警

---

## 关键公式

> 本文为系统论文，**原文不含任何编号公式或数学推导**。以下记录论文中明确定义的形式化对象与判定条件，均以原文描述为准，不虚构原论文中不存在的记号。

### 定义1: [[Safetime|安全时间]]

$$
\text{safetime} = \max \left\{ v \;\middle|\; \text{某分片范围上直到提交版本 } v \text{ 的所有变更均已被接收} \right\}
$$

**含义**：每个分片范围内「已确认收齐变更」的最高提交版本；它是判定变更能否安全释放、以及判定变更流是否完整的唯一依据。

**符号说明**：
- $v$：提交版本（commit version），即 Spanner 的提交时间戳
- 该最大值由常规**心跳**推进，但在被 blocked shard 消息覆盖时被压制在事务提交时间的下界

### 定义2: [[ChangeStream Filter|区间表示]]

查询 `where sender = "Carlos" and time > 2000` 被表示为 `["sender", "time"]` 条目下的区间：

$$
\big((\text{"Carlos"},\, 2000),\ (\text{"Carlos"},\, \infty)\big)
$$

**含义**：把 filter 条件编码为字段值元组上的一段区间，使「文档是否命中该查询」变成「文档的字段值元组是否落在区间内」，从而可用区间树做对数级检索。

**符号说明**：
- 区间下界为各等值字段的常量值与关系运算符的界
- 单个 filter 中**至多一个关系运算符**（其余必须是等值或 `in`），这正是区间可精确表示的前提

### 定义3: [[ChangeStream Filter|匹配集合的差集]]

设 $M(\cdot)$ 为某文档映像命中的查询集合，$A$ 为 post-image、$B$ 为 pre-image，则：

$$
\text{新命中} = M(A) \setminus M(B), \qquad \text{不再命中} = M(B) \setminus M(A)
$$

**含义**：一次变更只需求两次匹配再做集合差，即可同时得到「新进入结果集」与「离开结果集」的查询，避免了为删除类变更单独设计索引路径。

### 定义4: [[Bloom Filter|轮询校验条件]]

设 $n_{add}$ 为变更通知带来的新增结果数，$d$ 为 SDK 依据 Bloom filter 判定「确定已不在结果集」的文档数，$\Delta n$ 为结果总数的变化量，则删除被**精确确定**当且仅当：

$$
n_{add} - d = \Delta n
$$

**含义**：满足该等式时 SDK 可无歧义地推出哪些文档被删除；否则必须把查询 $Q$ 整体作废并透明重跑——这是轮询模式下保证一致性的兜底机制。

**符号说明**：
- $d$：由 Bloom filter「可能包含」的假阳性特性决定，$d$ 是**确定**不在结果集中的文档数（下界），而非全部删除数

### 定义5: [[500/50/5 规则|合规流量增长约束]]

$$
q(t + 5\,\text{min}) \le 1.5 \cdot q(t), \qquad q(0) = 500\ \text{QPS}
$$

**含义**：Firestore 要求客户端流量按此速率渐进爬升，给自动扩缩（[[Slicer]] 重新分片、Autopilot 扩容）留出反应时间；违反该规则的流量视为不合规，可能被限流或削减。

---

## 关键图表

### Figure 1: Firestore architecture / Firestore 总体架构

![[assets/Firestore/Firestore_fig1_architecture.png]]

**说明**：出自信任后端（Server SDK）与终端用户（Mobile / Web SDK）的请求经 **Firestore API** 进入，向下分为 **Frontend** 与 **Backend** 两个红色组件，二者均与 **Metadata cache**（蓝色）和 **Realtime Cache**（红色大块，即本文主角实时栈）交互，Backend 最终落到 **Spanner**。图中箭头只表示**请求**流向，不表示响应流。

### Figure 2: The Real-time Stack with ChangeLog and QueryMatcher tasks organized into groups / 实时栈的组结构与分片

![[assets/Firestore/Firestore_fig2_realtime_stack.png]]

**说明**：以 `select * from /room/C/messages where time > T order by time desc` 为例。两个 **Frontend** 各自持有查询，交叉扇出到绿/蓝两组 **QueryMatcher**（多个任务），每组下挂一个 **ChangeLog**，ChangeLog 再对接多层 **Backend**。右侧显示 key 空间的切分：`[/room/C/messages/a, room/C/messages/g)`、`[/room/C/messages/g, /users/gg)` 归蓝组，`[/users/gg, /user/zz)`、`[/user/zz, ...]` 归绿组。**一个 ChangeLog 喂多个 QueryMatcher**，多组共同覆盖整个 keyspace。

### Figure 3: ChangeStream Filter / 变更流过滤器结构

![[assets/Firestore/Firestore_fig3_changestream_filter.png]]

**说明**：图例中圆形为 [[Trie]] 节点（对应一个字段），三角为 [[Interval Tree]]，虚线方框为查询 ID 集合。示例 trie 的路径 `sender → time` 下挂区间树，其区间如 `{("Carlos", 2000), ("Carlos", ∞)}` 指向查询 `{7}`；`{("Alice",1000), ("Alice",∞)}` 与 `{("Carlos",3000), ("Carlos",∞)}` 分别指向 `{13,21}` 与 `{2}`；更深的 `{("Alice",-∞), ("Alice",1000)}` 指向 `{9}`。这直观展示了「同一集合上的查询通常共享字段组合、仅取值不同」这一前提。

### Figure 4: Successful two-phase interaction / 两阶段 Prepare-Accept 交互

![[assets/Firestore/Firestore_fig4_two_phase.png]]

**说明**：四条泳道从左到右为 **Backend / Spanner / ChangeLog / QueryMatcher**，八个步骤依次为：(1) Prepare Request（含 document keys）→ ChangeLog；(2) Prepare Response（min/max commit_time bounds）→ Backend；(3) Blocked Shard（min_commit_time, shard range）→ QueryMatcher；(4) Commit Request（时间界 + mutation）→ Spanner；(5) Commit Response（commit_time）→ Backend；(6) Accept（commit_time + 前后文档快照）→ ChangeLog；(7) Data Change → QueryMatcher；(8) Unblocked Shard → QueryMatcher。注意 **ChangeLog 只对上游做 (3)(7)(8)，Spanner 与 ChangeLog 之间无直接调用**。

### Figure 5: Safetime flow example / 安全时间流转示例

![[assets/Firestore/Firestore_fig5_safetime_flow.png]]

**说明**：一张 11 步的时间轴表格，列分别为 **Step / Action / Message from Changelog / Frontend buffer / Safetime**。集合 `/room/firestore_community/messages` 由 **shard 1** 与 **shard 2** 共同服务。关键过程：t0 心跳使查询 safetime 为 t0；txn1 写文档 A 到 shard 1（min_commit_time=t1）触发 `Blocked shard 1 (t1)`；txn2 也写 shard 1 但被更低的界吸收（无新消息）；txn3 写文档 C 到 shard 2 触发 `Blocked shard 2 (t20)`；第 7 步心跳给出 shard 1 的 t25 与 shard 2 的 t35，**最小心跳 safetime 取 t25**；txn1 在 t5 提交后 Frontend 缓冲区收到 A(t5)，但 safetime 为 t9 < t5 尚不能释放；直到第 9 步 txn2 在 t30 提交并 `Unblock shard 1 (commit time t30)` 后，A(t5) 才被 **Released**。这张表把「为什么必须等最小公共 safetime」讲得最清楚。

### Figure 6: Number of messages delivered and the number of active connections / 消息投递量与活跃连接数

![[assets/Firestore/Firestore_fig6_messages_connections.png]]

**说明**：2000 秒实验。左轴为累计消息数（红色 `time message` = 时间推进类消息，黄色 `data message` = 数据变更消息），右轴为活跃连接数（蓝线）。蓝线平滑上升至 ~100,000 连接，说明**所有新建查询都立即从 polling 切入 streaming 并一直保持**；堆叠面积显示消息投递总量随时间线性增长，未见限流或过载迹象。

### Figure 7: Notification latency / 通知延迟

![[assets/Firestore/Firestore_fig7_notification_latency.png]]

**说明**：p50（红）稳定在 ~40ms，p90（黄）~50ms，p99（绿）~110–140ms，**三条曲线在连接数与实时查询数增长 100 倍的过程中保持平稳**，这是实时栈透明扩缩的最直接证据。延迟定义为从 Spanner 确认提交，到所有已连接客户端收到对应通知。

### Figure 8: ChangeLog group count vs write throughput for the Bully database / Bully 库的写吞吐与 ChangeLog 组数

![[assets/Firestore/Firestore_fig8_changelog_groups.png]]

**说明**：对 us-east4 的 "Bully" 库施加非合规写负载，从 0 加速爬升到 **45k 写/秒**（加速度 183 writes/sec²）。绿色 `Allowed` 与红色 `Throttled` 堆叠表示被允许与被限流的写。蓝点线为 ChangeLog 副本组数：初始 1 个组，约 **5 分钟后**扩张到 **9 个组**；在这 5 分钟窗口内 Allowed 保持平坦（约 10k/s）而其余被限流，容量补齐后限流停止、所有写成功。5 分钟的调整节奏与 **500/50/5 规则**对齐。

### Figure 9: Throttled writes operation to the Bully vs all other databases in that region / 限流影响范围隔离

![[assets/Firestore/Firestore_fig9_throttle_isolation.png]]

**说明**：蓝线为 Bully 库的被限流请求数（峰值约 3500），黄线为同 region 其他所有客户库——**全程为 0**。这直接证明限流只作用于违规库，其他库的延迟指标也保持不变，即[[Performance Isolation|性能隔离]]成立。

### Figure 10: Task restart rate of QueryMatchers and impact on percentage of queries in polling mode / 任务重启率与轮询查询占比

![[assets/Firestore/Firestore_fig10_restart_polling.png]]

**说明**：横轴是 rollout 开始后的秒数（0 → 7200）。黑线为每分钟任务重启数，红点为处于 polling mode 的查询百分比。重启率峰值约**每 17 秒一次**，轮询查询占比短暂冲到 **31%**。作者指出完整 rollout 比随机故障**更糟**（所有任务在两小时内被重启一遍），因此这是一个偏保守的悲观测试。

### Figure 11: Notification latency (log scale) during a two hour rollout of QueryMatchers / 滚动发布期间的通知延迟

![[assets/Firestore/Firestore_fig11_latency_rollout.png]]

**说明**：纵轴为对数刻度。即使在 Figure 10 那样的重启风暴下，**中位通知延迟基本不变**，p90/p99 的抖动也在**不到一分钟**（系统的监控间隔）内恢复，因为被退回 polling 的查询很快重新进入 streaming mode。这是可用性论证的核心证据。

### Table 1: Notification latency breakdown / 通知延迟分解

| 阶段 | p50 | p95 |
|---|---|---|
| Commit 到消息被 ChangeLog 接受 | 3 ms | 30 ms |
| Commit 到消息抵达 QueryMatcher | 10 ms | 52 ms |
| ChangeStream Filter 执行 | 12 μs | 75 μs |
| Commit 到消息抵达 Frontend | 21 ms | 100 ms |
| Commit 到 Snapshot 发出 | 15 ms | 125 ms |

**表格说明**：单 region 部署实测。注意两点：(1) **ChangeStream Filter 执行只要 12μs / 75μs**，匹配本身几乎不构成瓶颈，延迟主要花在网络与排队上；(2) 「Commit 到 Snapshot 发出」（15ms）**低于**「Commit 到消息抵达 Frontend」（21ms）这一反直觉现象，论文解释为 Frontend 的扇出效应——前者以高扇出场景为主，而高扇出场景通常是小体积的低延迟通知。表中**不含**消息在 Google 网络之外送达终端设备的时间。

---

## 实验

### 生产负载特征（Section 8.1）

| 指标 | 数值 |
|---|---|
| 可用 region 数 | 45（最大为 us-central1，Iowa） |
| 数据库总数 | 800 万+ |
| 月度活跃终端用户 | 15 亿 |
| 使用实时查询的活跃库占比 | 69% |
| 通过实时查询 API 返回的结果占比 | 21% |
| 实时通知中新增/更新文档占比 | 94%（其余 6% 为删除或不再匹配） |
| 平均一次写触发的实时消息数 | 2.1 |
| 扇出最大的 10 个库的写读比 | 平均 1 : 12,000，峰值约 1 : 100,000 |

### 实验设置

- **环境**：us-east4 region 的生产 Firestore 数据库，实验在日平均生产负载时段进行
- **规模定位**：实验负载大于 99.99% 的活跃生产库，但仍比最大的生产库小一个数量级

#### 实验一：查询可扩展性（Figure 6 / Figure 7）

| 参数 | 值 |
|---|---|
| 聊天室数量 | 500 |
| 新建应用实例速率 | 50/秒（即 50 新连接、100 新实时查询每秒） |
| 每实例查询数 | 2（本室消息 + 被 @ 消息） |
| 峰值并发连接 | 100,000 |
| 峰值实时查询 | 200,000 |
| 每聊天室写速率 | 2 变更/秒（1 条新消息 + 1 次 metadata 更新） |
| 每聊天室终态规模 | 2000 消息文档 + 1 个 metadata 文档 |
| 测量时长 | 2000 秒 |

**结果**：所有查询立即切入 streaming 并保持；通知延迟 p50/p90/p99 全程平稳（Figure 7）。**polling → streaming 的切换延迟** p50 从实验开始时的 130ms 升到结束时的 260ms，作者归因于结果集随消息累积而变大（轮询查询耗时增加），但**切换本身的耗时并不随连接数与查询数增长**。

#### 实验二：写可扩展性与性能隔离（Figure 8 / Figure 9）

见上文 Figure 8、Figure 9 说明。核心结论：单库突发到 45k 写/秒时，系统通过约 5 分钟的重新分片（1 组 → 9 组）吸收，期间只限流该库，对其他库零影响。

#### 实验三：可用性（Figure 10 / Figure 11）

对 QueryMatcher 做两小时滚动发布（不做任务退出前的移交）。结果：重启率峰值 ~1 次/17 秒，轮询查询占比峰值 31%，但中位通知延迟不变、p90/p99 抖动一分钟内恢复。

### 可复现性评估

- [ ] 代码开源 —— 否，Google 内部生产系统
- [ ] 预训练模型 —— 不适用
- [x] 实现细节完整 —— 协议步骤、数据结构、分片比例（1:3）、调优节奏（5 分钟）均有披露
- [ ] 数据集可获取 —— 否，使用生产流量与自建合成负载
- 生产统计数字（8M 库 / 1.5B MAU / 69% / 21%）无法外部核验

---

## 批判性思考

### 优点

1. **对「不完整」的诚实处理**：整个设计的亮点不是「永不出错」，而是**显式建模「变更流可能不完整」**并给出确定性的降级路径（退回 polling mode 直到读时间越过不完整区间）。这比许多只描述正常路径的系统论文更接近工程现实。
2. **匹配索引与查询能力的协同设计**：把「至多一个关系运算符」这一限制当作**设计前提**而非妥协——正因为 filter 表达能力受限，才能用区间树精确表示；能力增强会让 $O(\text{matches}) + O(\log \text{queries})$ 退化。这种「能力换延迟」的取舍讲得很清楚。
3. **两阶段协议的非 2PC 性质分析到位**：明确指出「ChangeLog 失败不回滚 Spanner」并说明恢复靠上层降级，避免了把这个问题伪装成分布式事务问题。
4. **时序表格（Figure 5）质量极高**：逐步列出每条消息、Frontend 缓冲区状态与 safetime 的最小值计算，读者可以手推验证「为什么 A(t5) 必须等到 t30 之后才释放」——这类论文里少见的可验证性。
5. **诚实披露反模式**：明确承认「大文档上的实时查询」是反模式，也承认早期基于文档数的资源估计失效、被锁驱动的编程模型 bug 频发，这些负面经验比成功叙述更有价值。

### 局限性

1. **无编号公式、无形式化规范**：协议的正确性依赖自然语言描述；不完整流的处理、pool change 协议（明确因篇幅省略）等关键部分缺乏可验证的规范，读者难以独立论证其安全性。
2. **实验规模与生产规模有断层**：作者自陈实验负载**小于最大生产库一个数量级**，而扇出最大的库写读比达 1:100,000。图 6/7 的「平稳」能否外推到该量级并未直接验证。
3. **读写扩展的耦合点被淡化**：ChangeLog 位于写路径的 Prepare 阶段，论文承认「成功触达 ChangeLog 是每次写的额外要求」，虽然用故障域对齐论证它不引入新故障场景，但这仍是一个**同步写路径依赖**，其成本在论文中没有定量分析（例如 Prepare 往返对写延迟的贡献）。
4. **1:3 的 ChangeLog:QueryMatcher 静态比例**：这是按当前生产需求硬编码的，论文只说第 9 节提到可自定义，但未给出该比例如何影响延迟/成本，也没给出自适应方案。
5. **缺少与其他实时查询方案的横向实验对比**：相关工作（InvaliDB、RethinkDB 等）只做定性讨论，没有在同一负载下对比，因此「更好」的论断缺乏定量支撑。
6. **关键常数（限流阈值、宽松资源约束的具体值）未公开**：只说「由测试与生产流量实验确定」，可复现性受限。

### 潜在改进方向

1. **为「变更流不完整」给出形式化规范**：把 safetime 推进、blocked/unblocked、不完整区间恢复写成可机械检验的状态机，或做 TLA+ 验证。
2. **扇出的自适应调节**：把 1:3 的静态比例改为基于查询匹配代价与连接数的在线调节；ChangeStream Filter 的 $O(\text{matches})$ 特性使其代价可预测，适合作为控制信号。
3. **大文档实时查询的规避机制**：既然论文已识别该反模式，可在 API 层引入「仅推送变更字段的哈希/摘要」模式，避免传播完整 before/after 文档。
4. **Prepare 阶段的合并/批量优化**：把同一 Backend 上多个事务的 Prepare 批量发送（论文提到逻辑消息已按网络优化尺寸批量打包，但未说明是否跨事务合并时间界请求）。
5. **考虑查询能力的有损扩展**：例如支持范围排序但以「结果集失效重跑」为代价，把当前「精确表示」的边界有条件地放宽。

### 可复现性评估

论文的价值主要在**设计思想与工程经验**，而非可复现的实验。协议步骤、数据结构、组内比例与调优节奏都写得足够具体，理论上可据此构建一个原型；但真实的一致性保证依赖 [[Spanner]] + [[TrueTime]] + [[Chubby]] 这一整套 Google 内部基础设施，学术界难以照搬。生产统计数字全部为自报，无法外部验证。

---

## 关联笔记

### 基于

- [[Firestore|Firestore (ICDE 2023)]]: 本文的姊妹篇，给出 Firestore 全组件总览与实时查询的简要描述，本文是对其中实时查询部分的展开
- [[Spanner]]: 提供持久化、事务与强一致提交时间戳
- [[TrueTime]]: 提供提交时间的上下界，是 Prepare 阶段返回时间界的基础
- [[Chubby]]: ChangeLog 副本组的 leader 选举服务
- [[Slicer]]: 写负载驱动的 keyspace 自动切分/合并框架
- [[Firebase Realtime Database]]: 设计灵感来源（2014 年随 Firebase 被 Google 收购）

### 对比

- [[InvaliDB]]: 最接近的学术方案，同为写分片 + 查询副本扩展，但仅最终一致且分片疑似手动
- [[RethinkDB]] / Meteor / Parse: 开源实时查询实现，把整库变更流推到单节点，写扩展受限
- [[MongoDB]] Realm / Couchbase Lite: 提供本地库 + 实时查询 + 自动同步，但无强一致快照且要求本地复制整库
- [[NiagaraCQ]]: 同样对相关查询分组以高效共同执行，但面向变化较慢的外部数据源

### 方法相关

- [[ChangeLog]]: 把已提交写操作转换为有序变更流的核心组件
- [[QueryMatcher]]: 用 ChangeStream Filter 把变更匹配到活跃查询
- [[ChangeStream Filter]]: 字段列表 → 区间树的内存索引
- [[Interval Tree]]: 按区间下界排序并增广子树最大上界的 AVL 树
- [[Trie]]: 复用字段列表前缀的索引组织方式
- [[Safetime]]: 判定变更可释放性与流完整性的核心量
- [[Global Snapshot]]: 保证同一连接上多查询一致视图的消息
- [[Polling Mode]] / [[Streaming Mode]]: 实时查询的两种状态及降级路径
- [[Two-Phase Commit]]: 本文的 Prepare/Accept 与它形似而神不同（存储不回滚）
- [[Bloom Filter]]: 轮询模式下校验结果集删除的辅助结构
- [[Materialized Views]] / [[Incremental View Maintenance]]: 实时查询与物化视图在增量维护上的异同——Firestore 实时查询不属于任何事务作用域，因此允许跳过部分更新、直接从当前时间重跑
- [[Continuous Query]]: 连续查询范式，Firestore 属于其工程化实例
- [[Actor Model]]: 实时栈采用的编程模型
- [[Performance Isolation]] / [[Noisy Neighbor]]: 多租户架构下的核心挑战
- [[Isolation Pool]]: 应对异常流量的运维手段
- [[500/50/5 规则]]: 合规流量增长约束
- [[Collection Group Query]]: `/**/` 语法，跨同名集合读取
- [[Document Database]]: Firestore 的数据模型定位

### 硬件/数据相关

- [[LevelDB]]（iOS）/ SQLite（Android）/ IndexedDB（浏览器）: SDK 端本地缓存的三套实现
- Monarch: Google 的全球级内存时序数据库，用于监控告警

---

## 速查卡片

> [!summary] The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)
> - **核心**: 在 Spanner 之上构建无持久化 changelog 的实时查询栈，把「提交 + 时间推进」重组成强一致变更流
> - **方法**: [[ChangeLog]]（两阶段 Prepare/Accept + [[Safetime]] 心跳）+ [[QueryMatcher]]（[[ChangeStream Filter]] 区间树匹配）+ [[Frontend]]（提交序缓冲 + [[Global Snapshot]]）+ SDK 两区缓存与推测执行
> - **结果**: 20 万实时查询 / 10 万连接下通知延迟 p50 ~40ms、p99 ~120ms 全程平稳；匹配执行仅 12–75μs；单库 45k 写/秒仅 5 分钟即扩容吸收且不影响其他库；2 小时滚动发布中位延迟不变
> - **规模**: 45 个 region、800 万数据库、15 亿 MAU、69% 的库使用实时查询
> - **代码**: 无开源

---

*笔记创建时间: 2026-09-24*
