---
title: "VikingMem: A Memory Base Management System for Stateful LLM-based Applications"
method_name: "VikingMem"
authors: [Jiajie Fu, Junwen Chen, Mengzhao Wang, Aoxiang He, Maojia Sheng, Xiangyu Ke, Yifan Zhu, Yunjun Gao]
year: 2026
venue: PVLDB
tags: [llm-memory, memory-base, vector-database, rag, agent-memory, event-entity-model, long-term-memory]
zotero_collection: VLDB/2026
image_source: online
arxiv_html: https://arxiv.org/html/2605.29640
created: 2026-10-10
---

# 论文笔记：VikingMem: A Memory Base Management System for Stateful LLM-based Applications

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | 浙江大学；字节跳动（ByteDance） |
| 发表 | PVLDB 19(12), 2026, pp. 4344–4357（VLDB 2026） |
| DOI | [10.14778/3827998.3828037](https://doi.org/10.14778/3827998.3828037) |
| 对比基线 | [[Mem0]]、[[Mem0-graph]]、[[Zep]]、RAG、[[Mirix]]、Full-Context、OpenClaw、Claude Native Memory |
| 适用场景 | 社交陪伴、搜索推荐、效率协作、个性化教育、[[Agent Memory|自主智能体记忆]] |
| 链接 | [arXiv](https://arxiv.org/abs/2605.29640) / [Code](https://github.com/BytedanceFu/VikingMem) / [Use Cases](https://github.com/FuJiaJie123/VikingMem) |
| 开源子集 | [[OpenViking]]（面向 AI Agent 的开源 Context Database） |

---

## 一句话总结

> VikingMem 提出 [[Memory Base]] 这一数据管理范式，用可配置的 [[Event-Entity Model|事件-实体模型]] + 单次 LLM 推理抽取 + 算子化状态演化，统一服务多类 LLM 记忆应用，检索效果比基线最高提升 30%、同时保持毫秒级低延迟。

---

## 核心贡献

1. **形式化 Memory Base 范式**：提出三条核心原则——选择性抽取、内在状态性与演化、可泛化的抽象设计——作为 LLM 应用持久状态管理的新架构框架（§1）。
2. **设计并实现 VikingMem**：构建于 [[VikingDB]] 向量引擎之上的端到端 Memory Base 管理系统，核心技术模块包括智能记忆分段、算子驱动的实体更新机制、可配置抽取 schema（§3）。
3. **五大工业场景落地**：在字节跳动内部落地社交陪伴、搜索推荐、效率协作、教育、Agent 记忆五类场景（§4）。
4. **SOTA 效果与低延迟**：在 [[LOCOMO]] 与 [[LongMemEval]] 公开基准上，检索准确率（LLM-Judge Score）最高领先基线 30%，同时维持所有对比方法中最低的 P95 检索延迟（§5）。

---

## 问题背景

### 要解决的问题

LLM 应用正从无状态、短查询转向长时间、跨会话的有状态交互（持久助手、自主智能体、领域聊天系统）。LLM 的[[Context Window|上下文窗口]]虽在扩展，但始终是**有限、昂贵、延迟敏感**的资源：简单地把越来越长的历史塞进每次提示既不实用，也无法提供结构化、生命周期感知的状态管理（巩固、遗忘、溯源）。论文举例：一个陪伴学生数月的写作辅导助手，若长期状态在应用层以日志、浅缓存、任务专属启发式的方式零散管理，就会反复问重复问题或自相矛盾——这不是 LLM 推理能力的问题，而是**持久状态缺乏统一记忆机制**导致的碎片化设计，带来连续性断裂、记忆过期或冲突、用户信任下降，以及大量自定义协调逻辑的开发成本。

### 现有方法的局限

- **基于知识图谱的系统（ZEP、Mem0 的图集成）**：动态构图带来显著的构建时延迟，在生产环境单租户每日生成超 10 亿 token 记忆数据的高吞吐写入场景下不可用。
- **窄垂直领域系统（如 Memobase）**：针对对话用户画像这种单一场景刚性设计，无法容纳自主智能体所需的程序化 [[SOP|标准操作流程]]——存在**适配性缺口**：为聊天历史优化的 schema 无法迁移到 Agent 的流程追踪。
- 二者共同的问题是：行业缺乏一个用**单一基础设施**同时服务多元场景的统一、可配置记忆系统。

### 本文的动机

作者基于字节跳动服务多元企业客户与商业化 LLM 方案的经验，提出三条设计原则（对应三个产业观察）：

1. **低信噪比的原始流**（Observation 1）→ 必须做[[Selective Extraction|选择性信息抽取]]，而非粗暴分块或摘要，否则造成 "context pollution"（LLM 被无关噪声分散注意力或 "over-squashed"）。
2. **状态的动态本质**（Observation 2）→ 传统"插入-检索"式静态向量存储只会积累历史、不更新底层状态，导致矛盾与过期信息；VikingMem 需具备[[Statefulness and Evolution|内在状态性与演化]]能力，模拟人类记忆巩固过程。
3. **记忆架构的碎片化**（Observation 3）→ 不同下游应用（情感陪伴 vs 自主智能体）需要截然不同的记忆结构与内容；为每个场景单独建设系统维护成本高、无法共享能力，因此需要[[Generalizable Design Paradigm|可泛化的设计范式]]。

---

## 方法详解

### 核心数据抽象：Event-Entity 模型

[[Memory Base]] 的架构核心建立在两个基础数据抽象之上：**Event（事件**）与 **Entity(实体)** 。

#### Event

事件是记忆的基本单元，代表一条离散的、带时间戳的、情景式（episodic）记录。事件不是基于时间近因的原始上下文窗口，而是从可能交织的原始会话中**选择性抽取**出来的、语义连贯的、schema 约束的记忆单元（人类记忆本质上是基于事件、组织为离散情景记录的）。

#### Entity

实体代表一个持久的、不断演化的状态表示（例如完整的用户画像、Agent 的工具使用知识库）。与瞬时的事件不同，实体通过整合与巩固信息来形成连贯的长期记忆。实体不仅是压缩的状态笔记，而是通过显式的[[Aggregate Expression|聚合表达式]]和算子（Operator）从事件**增量物化**而来的持久状态表示——每个属性的 `AggregateExpression` 显式规定了属性更新规则：需要哪种 `EventType`、从传入事件取哪个 `PropertyName`。

#### Operator（算子）

算子是应用定义的函数，决定一个实体的状态如何被关联事件更新。当一条链接事件被摄入时，`AggregateExpression` 的 `Op` 字段指定的算子被执行，使用指定的事件属性来更新实体对应属性，完成记忆的更新演化周期。内置算子分两类：

- **统计类**（不调用 LLM）：`SUM`、`MAX`、`AVG`、`COUNT`，用于直接数值计算（如计算用户在练习题中的正确率），规避 LLM 在算术任务上的已知局限。
- **LLM 类**：`LLM_MERGE` 用 LLM 执行增量更新，处理内容去重与新旧信息冲突消解；`TIME_COMPRESS` 实现长期记忆生命周期管理（见下文"渐进式巩固"）。

### 范式解构：Event-Entity 作为通用算子代数

传统记忆架构本质上只有一种短生命周期、事件级（episodic）记忆概念；额外的"记忆类型"通常不是新抽象，而是（i）更换事件抽取用的 prompt，或（ii）把任务逻辑硬编码进 prompt——每个新场景都要再做一层 prompt 工程，记忆系统本身不提供稳定可复用接口。

VikingMem 把事件存储视为通用日志、实体存储视为该日志之上的一组物化视图（materialized view），事件与实体的关系表达为一条参数化查询：

```
entity := SELECT OP(event.content) FROM Events
          WHERE filters(event) GROUP BY keys(event)
```

其中 `keys` 定义事件如何被分组为实体（如按用户级、用户-助手对级、按主题级），`filters` 约束哪些事件参与（如时间窗口），`OP` 从固定算子库中选取。应用专属的 prompt 只出现在流水线的边缘（事件抽取、以及必要时的 LLM 算子内部），整体转换模式保持不变。只改变 `keys`、`filters` 与选定算子，同一事件日志就能物化出用户画像、SOP、Agent 工具工作记忆、搜索-推荐-问答轨迹，或学科级学习进度记录，而无需修改核心记忆引擎——这把记忆设计从逐 prompt 工程提升为一套在事件与实体之上的小型可复用代数。

### 系统设计目标

VikingMem 聚焦两个核心目标的权衡：

- **①记忆有效性**：给定查询 $Q$，系统检索或构造能最大化 $\mathcal{L}(Q, M)$（准确性、连续性、相关性）的记忆上下文 $M$。
- **②操作效率**：降低记忆生命周期的成本，包括从低密度流中识别重要事件的抽取成本，以及获取、重排、整合记忆进提示的检索延迟。

### 3.1 Memory Extract（记忆抽取）

抽取模块接收一个会话（一批 $N$ 条消息），当累积消息数达到阈值（$\geq 20$ 条）即触发，把累积数据转化为持久长期记忆，防止 LLM 活跃上下文过载而导致幻觉与计算复杂度升高。短期记忆定义为用户活跃会话内最近数据（保留以供即时上下文相关性），同一会话随后转为持久长期记忆。

#### One-pass Memory Extraction（单次抽取范式）

传统范式为每种记忆类型定义独立、硬编码的 prompt，要抽取 $k$ 种记忆类型就要把同一原始输入喂给 LLM $k$ 次，带来过高的 token 消耗和计算成本。VikingMem 把可配置的 [[Event-Entity Model|事件与实体 schema]] 编译为**单一、综合的 prompt**，LLM 只处理一次原始输入流，在同一次推理中**同时**抽取所有定义的记忆类型——既包括新事件，也包括实体更新所需的精确内容。

#### Faster Entity Update：Entity Update Algorithm（EUA）

即使单次 LLM 调用能抽取所有记忆类型，当实体字段是 string 类型时更新实体仍可能需要额外一次 LLM 调用（把抽取出的事件连同旧实体一起喂给 LLM 来合成新实体）——这对离线处理可以接受，但在延迟敏感场景（实时对话内的知识/属性摘要）下，第二次 LLM 调用显著增加响应时间与成本。

VikingMem 受 [[Roocode]] 启发提出 **[[Entity Update Algorithm|EUA]]**，无需任何额外 LLM 即可更新实体：抽取器为 schema 中每个实体字段输出紧凑的 patch，形式为 `«« SEARCH ... ==== ... »» REPLACE`；EUA 用[[Edit-Distance-based Approximate Search|基于编辑距离的近似搜索]]在旧字段中定位最佳匹配跨度（鲁棒于轻微 LLM 不精确导致的精确字符串匹配失败），再用 REPLACE 内容替换，把实体维护变成一个轻量级的确定性后处理步骤。该过程不需要枚举所有已存储实体——每次更新先用 ANN 搜索在实体表示上检索一个小的候选集（部署中为 top-5），只把这些候选纳入 patch 生成的 prompt，从而在保留足够上下文的同时限制 prompt 长度。

#### Intelligent Memory Segmentation（智能记忆分段）

输入数据流（尤其是会话类）常呈现低信息密度与话题交织（如从主题 A 跳到 B 再跳回 A）。消息级存储产生碎片化记忆、会话级存储引入过多噪声、顺序主题级方法无法合并非连续片段。VikingMem 用 LLM 执行两阶段策略：

1. **语义显著性过滤**：识别并仅保留最有意义的片段，修剪对话填充物（如开场问候），这是相比传统 RAG 分块策略的显著优势。
2. **事件中心划分**：LLM 确定每个连贯主题的精确起止位置，以**坐标元组列表**的形式输出——这种格式精确又高度 token 高效，支持合并语义相关但非连续的对话片段。

示例：一段关于递归（主题 A）的对话被博弈算法讨论（主题 B）打断。碎片化检索（A1, B, A2）会产生不完整答案，会话级整合会引入 B 的噪声导致事实错误；VikingMem 能把 A1 与 A2 准确合并为统一记忆、排除 B 的噪声。

### 3.2 Memory Management（记忆管理）

包含三项核心功能：事件记忆管理、实体记忆更新、关键词图更新。

#### Event Memory Management

抽取新记忆后首先去重：同一会话内高度相似的记忆通过基于 LLM 的解析被识别与合并。

#### 渐进式巩固：TIME_COMPRESS 算子

`TIME_COMPRESS` 算子实现长期记忆生命周期管理的"渐进式巩固"设计——不把记忆当作扁平日志，而是把相关事件分组进**主题中心的时间线**（topic-wise timeline）并按时序排列，实现"懒合并"（lazy merge）策略：

- 近期记忆保持高保真度；
- 不活跃时间线中的旧事件只在老化进入预定义时间窗口（如每周或每月）后才被合成为高层摘要；
- 合成后，系统存储高层摘要并为底层细粒度事件分配 [[TTL|Time-To-Live]]；
- 一旦摘要已保留其显著信息，过期的低层事件可被安全剪除，除非后续召回重新激活该时间线。

#### Keyword Graph（关键词图）

默认混合检索（稠密+稀疏）对诸如"你还记得我的昵称吗？"这类查询处理不佳，因为直接查询与间接记忆之间的语义相似度很低。VikingMem 构建一个[[Keyword Graph|关键词图]]：每个关键词的 embedding 由包含该关键词的记忆片段 embedding 平均而来，更好地捕捉语义特征，关键词与关联记忆相连。

### 3.3 Memory Retrieve（记忆检索）

#### Multi-path Recall with Time-Decay and Business Weights

检索分为两条路径以提升效果：

- **主路径**：结合稠密语义匹配与稀疏词法匹配的混合检索，并在原始检索分之外融入两个显式先验——时间衰减分 $S_{\text{time}}$（近因性）和业务重要性分 $S_{\text{busi}}$。
- **辅助路径**：利用关键词图检索补充记忆，覆盖标准混合搜索可能漏掉的场景。

实证表明，直接合并两路径结果的效果不如**独立对每条路径排序、分配不同配额（主路径配额更大）、再合并重排后的选集**——这带来更均衡、更高质量的最终检索集。

$S_{\text{busi}}$ 支持两种互补定义：（i）**类型级权重**——为不同事件记忆类型分配不同重要性权重；（ii）**实例级权重**——在 schema 中加入专门的权重字段，随事件一起抽取，用于在同一类型内提升高价值实例的召回排名。

#### Multi-vector Rerank

检索场景对端到端延迟要求苛刻（重排操作通常要求 p99 延迟在数百毫秒量级）。[[mGTE]] 等基于交叉编码器的重排算法在候选集较大时 p99 延迟可达秒级，不适合实时应用。VikingMem 采用受 [[ColBERT]] 启发的重排策略，通过**上下文化晚期交互**（contextualized late interaction）实现高效有效的记忆搜索：在抽取阶段预计算并存储记忆的 ColBERT 向量，应用一系列压缩技术（包括[[Product Quantization|量化]]与 [[Token-Merge|token-merge]] 操作），使存储开销保持与稠密向量相当，同时以极小的额外存储成本实现高效重排。

---

## 关键公式

### 公式 1：[[Workload-Aware Pre-Warming|多路召回最终融合分]]（Multi-path Recall 最终得分）

$$
S_{\text{final}} = (1 - w_{\text{time}} - w_{\text{busi}}) \cdot S_{\text{origin}} + w_{\text{time}} \cdot S_{\text{time}} + w_{\text{busi}} \cdot S_{\text{busi}}
$$

**含义**：对每条候选记忆计算一个加权组合分数，综合"原始检索分"（语义/词法相关性）、"时间近因性"与"业务重要性"三类先验，驱动最终排序。

**符号说明**：

- $S_{\text{final}}$：候选记忆的最终融合分
- $S_{\text{origin}}$：归一化后的原始混合检索分（dense + sparse）
- $S_{\text{time}}$：时间衰减分——记忆年龄在用户可配置的新鲜度容忍窗口内（如 1 天）取 1，超出窗口后按"先快后慢"曲线指数衰减
- $S_{\text{busi}}$：业务重要性分——类型级权重或实例级权重两种定义之一
- $w_{\text{time}}, w_{\text{busi}} \in [0, 1]$：可配置权重，满足 $w_{\text{time}} + w_{\text{busi}} \leq 1$

> 注：实验部分由于两个数据集均以事实型查询为主、时间加权收益有限，默认关闭了该特性（即 $w_{\text{time}} = w_{\text{busi}} = 0$）。

---

## 关键图表

> 共 5 个 Figure、6 个 Table（主文正文 Table 1–6），全部按原论文编号完整保留。

### Figure 1: Event-Entity Schema and Operators / 事件实体 schema 与算子定义

![VikingMem_fig1_event_entity_schema](https://arxiv.org/html/2605.29640/event_entity_schema.png)

左侧为 **Event Schema**：`EventType`、`Description`、`Properties`（每个属性含 `PropertyName`、`PropertyType`、`Description`）。右侧为 **Entity Schema**：`EntityType`、`Description`、`Properties`，每个属性额外含 `AggregateExpression`（`Op`、`EventType`、`PropertyName`、`IsPrimaryKey`）。底部列出内置算子：`LLM_MERGE`、`SUM`、`MAX`、`AVG`、`TIME_COMPRESS`、`COUNT`。该图是整篇论文 [[Event-Entity Model|事件-实体模型]] 的定义性模板。

### Figure 2: The Pipeline of VikingMem / VikingMem 整体流水线

![VikingMem_fig2_pipeline](https://arxiv.org/html/2605.29640/pipeline.png)

展示从输入数据流（Session、历史消息、User Profile）经 Extract Module（System Instruction + Memory Schema 的固定 prompt 缓存 + Input Messages）产出 Event Memory 与 Entity Memory，存入 VikingDB 支撑的存储层（含旧事件的时间线压缩/TTL）；Retrieve Module 对 Long-term Memory 执行基于混合向量搜索的多路召回 + [[ColBERT|MutiVector]] 重排，叠加 Short-term Memory 与 Query 一起生成带记忆的回复。

### Figure 3: The Extract Paradigm in VikingMem / 抽取范式对比

![VikingMem_fig3_extract_method](https://arxiv.org/html/2605.29640/extract_method.png)

(a) 传统方法（Previous Method）：$N$ 种记忆类型各自对应独立 Prompt，输入数据被 $N$ 次喂给 LLM，产出 $N$ 组独立记忆。(b) 本文方法（Our Method）：Event Schemas 与 Entity Schemas 分别编译为 Event Prompt 与 Entity Prompt，输入数据经 LLM **一次**推理同时产出 Entity-related Events 与 Other Events；Entity-related Events 连同 Operators 作用于 Old Entities 产出 New Entities。该图直观对比了[[One-pass Memory Extraction|单次抽取]]相对多次抽取的结构性简化。

### Figure 4: Faster Entity Update w/o LLM / 无 LLM 的更快实体更新

![VikingMem_fig4_faster_entity_update](https://arxiv.org/html/2605.29640/patch.png)

左图（传统）：Event Prompt → LLM → Entity-related Events，再经 Operators 结合 Old Entities 产出 New Entities（Operators 步骤隐含一次额外 LLM 调用来合成文本）。右图（EUA）：Event Prompt 与 Entity Prompt 合并后，LLM 一次产出 Entity-related **Patch**，Operator（Patch 应用）直接作用于 Old Entities 产出 New Entities，无需第二次 LLM 调用。对应 **Algorithm 1: EUA: Patch-based Entity Update w/o LLM**：

```
Input: Entity schema S; old entity E_old; field-wise patches {p_f}
Output: Updated entity E_new
1  E_new ← E_old
2  foreach field f in S do
3      (s, r) ← ParsePatch(p_f)        // s=SEARCH, r=REPLACE
4      if s = ∅ then continue
5      (i, j) ← BestApproxSpan(E_old[f], s)   // min edit distance
6      E_new[f] ← E_old[f][0:i] ‖ r ‖ E_old[f][j:]
7  return E_new
```

### Figure 5: Use Case for Agent Memory / Agent 记忆用例

![VikingMem_fig5_agent_tool_memory](https://arxiv.org/html/2605.29640/tool_memory.png)

左侧 **Event Instance**：记录工具调用的 `tool`、`description`、`is_success`、`use_situation`、`token_usage`、`time_cost_sec`、`feedback`、`thoughts` 等字段及 `event_time`。右侧 **Entity Instance**：由事件 **Evolve** 而来的工具持久档案，含 `tool_name`、`description`、`tool_call_times`、`success_rate`、`avg_token_usage`、`avg_time_cost_sec`、`suitable_for`、`failure_cases`、`suggestions` 及 `last_updated`。展示原始工具调用日志如何被选择性抽取为离散事件、再演化为可复用的"经验教训"实体档案。

### Table 1: LOCOMO 与 LongMemEval 上的 LLM-Judge 分数与检索延迟

**LOCOMO（GPT-4o-mini）**

| Methods | Single Hop | Multi-Hop | Open Domain | Temporal | Overall | p50 (s) | p95 (s) |
|---|---:|---:|---:|---:|---:|---:|---:|
| Mem0† | 72.93 | 67.13 | 51.15 | 55.51 | 66.88 | 0.15 | 0.20 |
| Mem0-graph† | 75.71 | 65.71 | 47.19 | 58.13 | 68.44 | 0.48 | 0.66 |
| Zep† | 74.11 | 66.04 | 67.71 | 79.79 | 75.14 | 0.42 | 0.63 |
| RAG | 65.16 | 50.35 | 48.96 | 59.50 | 60.26 | 0.22 | 0.43 |
| Full-Context | 78.00 | 74.82 | 59.38 | 83.80 | 77.47 | / | / |
| Claude | 63.50 | 55.67 | 50.00 | 47.35 | 57.86 | 18.90 | 25.42 |
| Openclaw | 17.60 | 13.48 | 30.21 | 13.40 | 16.75 | 17.19 | 26.26 |
| Mirix | 79.08 | 76.01 | 66.67 | 80.86 | 78.66 | 10.90 | 25.76 |
| **VikingMem** | **94.89** | **81.91** | **78.12** | 82.24 | **88.83** | 0.20 | 0.39 |

**LOCOMO（GPT-4.1-mini）**

| Methods | Single Hop | Multi-Hop | Open Domain | Temporal | Overall | p50 (s) | p95 (s) |
|---|---:|---:|---:|---:|---:|---:|---:|
| Mem0† | 62.41 | 57.32 | 44.79 | 66.47 | 62.47 | 0.15 | 0.20 |
| Mem0-graph | 74.44 | 68.44 | 51.04 | 54.52 | 67.73 | 0.48 | 0.66 |
| Zep† | 79.43 | 69.16 | 73.96 | 83.33 | 79.09 | 0.42 | 0.63 |
| RAG | 69.56 | 56.74 | 57.29 | 64.17 | 65.32 | 0.27 | 0.91 |
| Full-Context | 88.70 | 77.30 | 72.92 | **92.21** | 86.36 | / | / |
| Claude | 72.29 | 68.44 | 56.25 | 55.45 | 67.08 | 10.81 | 18.62 |
| Openclaw | 26.52 | 18.09 | 26.04 | 21.81 | 23.96 | 13.73 | 19.26 |
| Mirix† | 85.11 | 83.70 | 65.62 | 88.39 | 85.38 | 10.90 | 25.76 |
| **VikingMem** | **93.46** | **85.89** | **79.79** | 88.16 | **90.12** | 0.20 | 0.39 |

**LongMemEval（SSU=single-session-user, MS=multi-session, SSP=single-session-preference, TR=temporal-reasoning, KU=knowledge-update, SSA=single-session-assistant）**

| LLM | Methods | SSU | MS | SSP | TR | KU | SSA | Overall | p50 (s) | p95 (s) |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| GPT-4o-mini | Mem0 | 68.57 | 41.35 | 20.00 | 37.59 | 66.67 | 5.36 | 42.80 | 0.40 | 1.00 |
| GPT-4o-mini | Mem0-graph | 72.86 | 38.35 | 16.67 | 34.59 | 60.26 | 35.71 | 44.00 | 1.10 | 2.09 |
| GPT-4o-mini | Zep† | **92.90** | 47.40 | 53.30 | 54.10 | **74.40** | 75.00 | 63.21 | 3.80 | 5.45 |
| GPT-4o-mini | Full-Context | 81.43 | 40.60 | 30.00 | 36.84 | 76.92 | 82.14 | 55.00 | / | / |
| GPT-4o-mini | **VikingMem** | 90.00 | **53.31** | 46.67 | **55.89** | 71.23 | **96.43** | **66.36** | 0.25 | 0.89 |
| GPT-4o | Mem0 | 80.00 | 50.38 | 26.67 | 42.11 | 74.36 | 10.71 | 50.20 | 0.40 | 1.00 |
| GPT-4o | Mem0-graph | 84.29 | 48.87 | 30.00 | 41.35 | 75.64 | 48.21 | 54.80 | 1.10 | 2.09 |
| GPT-4o | Zep† | **92.90** | 57.90 | 56.70 | 62.40 | **83.30** | 80.40 | 70.40 | 3.80 | 5.45 |
| GPT-4o | Full-Context | 81.43 | 44.36 | 20.00 | 45.11 | 78.21 | 94.64 | 59.20 | / | / |
| GPT-4o | **VikingMem** | **92.90** | **69.92** | 56.70 | **66.92** | 76.92 | **98.21** | **75.80** | 0.25 | 0.89 |

**说明**：VikingMem 在两个基准、两套 LLM Judge 设置下均取得最高 Overall 分数，且检索延迟（p50/p95）始终与 Mem0/Zep 等轻量方法同量级，远低于 Mirix 和 Claude Native Memory；Mirix 在 LongMemEval 上甚至因 24 小时时限超时未能完成评测。

### Table 2: Effect of One-pass Extraction and EUA / 单次抽取与 EUA 的效果

| Techniques | Cost($) | Time(s) | Score |
|---|---:|---:|---:|
| Multiple Prompts | 0.35 | 11.02 | 88.86 |
| One-pass (w/ EUA) | 0.07 | 7.42 | 88.77 |
| One-pass (w/o EUA) | 0.11 | 12.32 | 88.83 |

**说明**：单次抽取相比传统多次 Prompt 基线把抽取成本从 \$0.35 降到 \$0.11（3.2× 下降），分数几乎不变；EUA 进一步把端到端耗时从 12.32s 降到 7.42s、成本从 \$0.11 降到 \$0.07，质量保持相当。

### Table 3: Storage Efficiency on LongMemEval / LongMemEval 存储效率

| Method | Storage (Tokens) | Score |
|---|---:|---:|
| Naive RAG | 100% (Baseline) | 63.81 |
| **VikingMem** | **16.82% (83.18% ↓)** | **75.80** |

**说明**：VikingMem 只保留抽取后的事件与实体快照，把存储占用压缩到原始的 16.82%（约 6:1 压缩比），同时 LLM-Judge 分数仍优于 Naive RAG（75.80 vs 63.81），证明选择性保留在压缩存储的同时保留了任务相关信息。

### Table 4: Ablation Study on LOCOMO (GPT-4o-mini) / 消融实验

| Removed Component | LLM Judge Score | Search Latency Component |
|---|---:|---:|
| / (完整模型) | 88.83 | - |
| Multi-Vector Rerank | 85.19 | -6.8ms |
| Entity Memory | 86.93 | ≈ 0 |
| IMSM | 83.51 | ≈ 0 |
| Keyword Graph | 86.92 | -25.8ms |

**IMSM = Intelligent Memory Segmentation Method（智能记忆分段方法）**。

**关键发现**：移除 IMSM 造成最大降幅（88.83→83.51，降 5.32 分），是对效果影响最大的单一组件，且因是离线处理，不引入任何检索延迟（≈0）；重排次之（降 3.64 分，但只增加 6.8ms p95 延迟）；实体记忆与关键词图各降约 1.9 分，关键词图是延迟开销最高的模块（+25.8ms）。

### Table 5: Operator Usage Frequency Across Real-World Application Scenarios / 真实应用场景中的算子使用频率

| Scenario | SUM | MAX | AVG | COUNT | LLM_M | TIME_C |
|---|---:|---:|---:|---:|---:|---:|
| Education | 85% | 90% | 96% | 88% | 91% | 26% |
| Agent Memory | 73% | 96% | 98% | 64% | 98% | 42% |
| Social Comp. | 24% | 16% | 32% | 11% | 100% | 72% |

**说明**：统计来自 100+ 客户应用的实体 schema 定义（每客户平均 2.6 个 schema）。教育与 Agent 记忆场景偏好聚合类算子（MAX/AVG/LLM_MERGE），用于总结累积状态与执行统计；社交陪伴场景极少用数值聚合，但 `LLM_MERGE` 用于 100% 的 schema、`TIME_COMPRESS` 用于 72%，体现其对语义合并与时间压缩的强需求。

### Table 6: F1-score (%) on LOCOMO / LOCOMO 上的 F1 分数

| Methods | SH | MH | OD | Temp | Overall |
|---|---:|---:|---:|---:|---:|
| Mem0 | 47.65 | 38.72 | 28.64 | 48.93 | 45.09 |
| Mem0-graph | 49.27 | 38.09 | 24.32 | 51.55 | 46.14 |
| Zep | 49.56 | 35.74 | 41.37 | 52.04 | 47.03 |
| Full-Context | 55.64 | 43.52 | 40.43 | **58.32** | 53.03 |
| Claude | 41.23 | 34.23 | 28.06 | 46.32 | 40.19 |
| Openclaw | 20.12 | 11.36 | 10.04 | 21.32 | 18.14 |
| **VikingMem** | **59.59** | **44.52** | **43.13** | 55.62 | **54.98** |

**说明**：以统一答案生成模型（gpt-4o-mini）计算 token 级 F1，VikingMem 取得最佳总体 54.98%，优于 Full-Context（53.03%），在单跳、多跳、开放域问题上均最佳，与 LLM-Judge 结论一致，交叉验证了方法的有效性。

---

## 实验

### 数据集

| 数据集 | 规模 | 特点 | 用途 |
|---|---|---|---|
| [[LOCOMO]] | 10 组对话，平均每组 1000+ 消息 | 单跳、多跳、时间推理、开放域知识四类问答 | 长期对话记忆评测 |
| [[LongMemEval]]（_s 子集） | 500 组长对话，平均约 115,000 tokens（比 LOCOMO 长 346×） | 带时间戳的对话历史，多会话推理与记忆更新类问题 | 更长时程交互记忆评测 |

### 实现细节

- **硬件**：向量数据库服务部署于单节点（32 vCPU Intel Xeon Platinum 8582C + 8GB 内存）；Embedding 服务运行在单张 NVIDIA A30 GPU（24GB 显存）+ 16 CPU + 64GB 内存。
- **LLM 设置**：LOCOMO 用 GPT-4.1-mini 与 GPT-4o-mini；LongMemEval 用 GPT-4o-mini 与 GPT-4o；同一模型同时承担答案生成与 LLM-as-a-Judge 评测。
- **公平性措施**：每个基线调用其官方记忆接口完成评测；每个系统只导入一次数据（摄入成本高）；每次查询重复生成与评测三次取平均；实施 24 小时端到端超时，超时基线不计入结果（Mirix 在 LongMemEval 上即因此被排除）；默认关闭时间加权（因两数据集均为事实型查询，受益有限）。
- **消融实验设置**：针对单次抽取与 EUA 的量化分析，在 LOCOMO 上配置三种记忆类型（一个事件记忆 + 两个实体记忆：用户画像与主题压缩记忆），多次 Prompt 基线采用并行 LLM 调用以保证延迟对比公平。

---

## 批判性思考

### 优点

1. **范式层面的抽象贡献**：把"记忆系统"从一堆场景专属 prompt 工程，提炼为"事件日志 + 实体物化视图 + 算子代数"的通用查询式表达（`entity := SELECT OP(event.content) FROM Events WHERE filters(event) GROUP BY keys(event)`），这个类 SQL 的抽象给出了一个可迁移、可复用的记忆系统设计语言，而非单纯刷榜式的检索优化。
2. **工程细节扎实、面向生产约束**：EUA 用编辑距离近似搜索代替第二次 LLM 调用来做实体更新，是一个针对真实延迟/成本瓶颈的务实解法；单次抽取范式也直接解决了多类型记忆抽取的 token 爆炸问题，并给出量化证据（Table 2）。
3. **消融实验的归因清晰**：Table 4 把每个组件（Rerank、Entity Memory、IMSM、Keyword Graph）对效果与延迟的边际贡献都单独量化，IMSM 作为离线处理却贡献最大效果增益（零延迟代价）这一结论具有实际指导意义。
4. **大规模生产场景的算子使用统计（Table 5）**：这是少见的"真实客户 schema 统计"证据，直接支撑了"不同场景需要不同算子组合"这一设计动机，而非纯理论论证。

### 局限性

1. **时间加权特性未实际验证效果**：论文设计了精细的 $S_{\text{time}}/S_{\text{busi}}$ 融合公式，但在两个主实验基准上因"查询多为事实型"而默认关闭，缺少专门针对时间敏感查询的定量评测来验证该机制本身的增益。
2. **EUA 的失败模式未充分讨论**：基于编辑距离的近似匹配对"轻微 LLM 不精确"鲁棒，但当 SEARCH 片段在旧字段中有多个相近匹配、或字段发生较大结构性变化时，patch 定位可能出错；论文未给出 EUA 失败率或错误恢复机制的量化数据。
3. **关键词图的构建与维护成本未展开**：Table 4 显示 Keyword Graph 是延迟开销最大的模块（+25.8ms），但论文未说明其规模增长后的维护代价（如关键词数量、embedding 更新频率）。
4. **多数基线分数标注为"引用自已发表结果"（†）**：部分 LOCOMO/LongMemEval 基线分数并非作者自己复现测量，存在评测设置细节不完全可比的风险，尽管作者声明遵循了 Zep 的评测协议。
5. **VikingDB 作为底层存储的具体机制未展开**：论文多次提到"VikingDB 负责持久存储"，但对 VikingDB 本身的索引结构、分布式架构等细节基本未涉及，读者无法独立评估存储层对延迟数字的具体贡献。

### 可借鉴方向

- 若要设计跨场景复用的记忆/状态管理系统，"事件日志 + 算子化物化视图"的 SQL 式抽象比"为每个场景写 prompt"更具工程可维护性，值得推广到其他需要持久状态演化的 LLM 应用设计中。
- 延迟敏感场景下，用确定性的 patch/diff 应用（如编辑距离近似匹配）替代第二次 LLM 调用，是降低多轮 LLM 流水线尾延迟的一个通用思路，不限于实体更新场景。
- 消融实验应优先量化"离线预处理类组件"（如本文的 IMSM）与"在线检索类组件"（如 Rerank、Keyword Graph）对效果与延迟的不同贡献结构，便于确定优化优先级。

---

## 关联笔记

### 基于
- [[RAG]]：VikingMem 与传统 RAG 对比，论文强调记忆系统需要选择性抽取与状态演化，而非静态分块检索。
- [[Roocode]]：EUA 算法设计灵感来源，借用其 patch 格式思想实现无 LLM 的实体更新。

### 对比
- [[Mem0]] / [[Mem0-graph]]：代表性基线，分别为向量检索式记忆抽取（ADD/UPDATE/DELETE/NOOP）与知识图谱扩展版本。
- [[Zep]]：代表性基线，构建情景、语义、社区记忆层并做带时间信号的混合检索重排。
- [[Mirix]]：代表性基线，用六个专用记忆模块做跨类型混合检索，是 LOCOMO 上效果第二强但延迟高两个数量级的系统。
- [[ColBERT]]：VikingMem 多向量重排策略的灵感来源（上下文化晚期交互）。

### 方法相关
- [[Memory Base]]：本文提出的新架构范式，VikingMem 是其具体系统实现。
- [[Event-Entity Model]]：VikingMem 的核心数据抽象，统一事件（episodic）与实体（持久状态）两类记忆。
- [[Entity Update Algorithm]]：无需额外 LLM 调用的实体更新机制（EUA）。
- [[Keyword Graph]]：补充语义相似度不足场景的辅助检索路径。
- [[VikingDB]]：VikingMem 的底层向量存储与检索引擎。
- [[OpenViking]]：VikingMem 部分核心能力的开源版本，面向 AI Agent 的开源 Context Database。

### 硬件/数据相关
- [[LOCOMO]] / [[LongMemEval]]：VikingMem 主要评测使用的长期对话记忆基准数据集。

---

## 速查卡片

> [!summary] VikingMem
> - **核心**：提出 Memory Base 范式（选择性抽取 + 状态性演化 + 可泛化设计），VikingMem 是其在 VikingDB 之上的端到端实现。
> - **数据模型**：Event-Entity 模型，事件是离散情景记录，实体通过算子（SUM/MAX/AVG/COUNT/LLM_MERGE/TIME_COMPRESS）从事件增量物化为持久状态。
> - **抽取**：单次 LLM 推理同时抽取所有记忆类型（one-pass extraction）；EUA 用编辑距离近似匹配实现无 LLM 实体更新；两阶段智能记忆分段处理话题交织会话。
> - **检索**：混合稠密/稀疏多路召回 + 时间衰减/业务权重融合 + 关键词图辅助路径 + 受 ColBERT 启发的多向量重排。
> - **证据**：LOCOMO/LongMemEval 上 LLM-Judge 分数最高领先基线 30%，F1 分数同样最优；检索延迟 p50 0.20–0.25s、p95 ≤0.89s，与 Mem0/Zep 同量级但效果显著更优；相比 Naive RAG 存储压缩至 16.82%（83.18% 下降）且效果更优；单次抽取+EUA 相比多次 Prompt 基线成本降至 20%、耗时降至 67%。
> - **代码**：https://github.com/BytedanceFu/VikingMem

---

*笔记创建时间: 2026-10-10*
