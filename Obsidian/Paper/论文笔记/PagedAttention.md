---
title: "Efficient Memory Management for Large Language Model Serving with PagedAttention"
method_name: "PagedAttention"
authors: [Woosuk Kwon, Zhuohan Li, Siyuan Zhuang, Ying Sheng, Lianmin Zheng, Cody Hao Yu, Joseph E. Gonzalez, Hao Zhang, Ion Stoica]
year: 2023
venue: SOSP 2023
tags: [llm-serving, inference-optimization, gpu-memory, kv-cache, attention]
zotero_collection: ""
image_source: local
arxiv_html: ""
created: 2026-07-31
---

# 论文笔记：Efficient Memory Management for Large Language Model Serving with PagedAttention

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | UC Berkeley、Stanford University、UC San Diego、Independent Researcher |
| 日期 | October 2023 |
| 项目主页 | [vLLM](https://github.com/vllm-project/vllm) |
| 对比基线 | [[Orca]]、[[FasterTransformer]] |
| 链接 | [arXiv](https://arxiv.org/abs/2309.06180) / [Code](https://github.com/vllm-project/vllm) / [DOI](https://doi.org/10.1145/3600006.3613165) |

---

## 一句话总结

> PagedAttention 将 [[KV Cache]] 虚拟化为按需映射的定长块，使 vLLM 以近零浪费和块级共享提升 LLM 服务吞吐。

---

## 核心贡献

1. **量化 KV cache 内存浪费**：指出连续预分配同时造成预留空间、内部碎片和外部碎片；被测系统仅有 20.4%–38.2% 的 KV cache 空间存放真实 token state。
2. **PagedAttention**：让注意力内核通过 [[Block Table|块表]]访问非连续物理 KV 块，而非要求 KV cache 连续，保持注意力语义不变。
3. **vLLM 服务引擎**：以逻辑/物理块映射、[[Copy-on-Write|写时复制]]、引用计数、[[Prefix Caching|共享前缀]]、抢占与分布式调度承载多种解码算法。
4. **端到端验证**：相同延迟下，相对 Orca/FasterTransformer 获得 2–4× 常规吞吐提升；复杂 beam search 和长共享前缀的优势更明显。

---

## 问题背景

### 要解决的问题

[[Autoregressive Generation|自回归生成]]每步都要读取全部历史 [[KV Cache]]。KV cache 会随 token 增长、请求长度未知且生命周期异构；GPU 中模型权重固定后，KV cache 决定可以并发多少请求，进而决定服务吞吐。

以 OPT-13B 为例，一个 token 的 KV cache 为 $2\times5120\times40\times2\text{ bytes}=800\text{ KB}$；2048-token 请求最多需要 1.6 GB。若必须为每条请求预留其最长序列的连续 buffer，批大小很快受限。

### 现有方法的局限

[[Orca]] 等迭代级 batch 系统仍将每个请求的 KV cache 存成连续 tensor，并为最大长度预分配：

- **Reservation**：尚未生成 token 的空间提前被独占；
- **内部碎片**：真实输出短于最大长度；
- **外部碎片**：大小不一的连续分配使 allocator 留下不可用洞；
- **不能共享**：平行采样、[[Beam Search|束搜索]]的公共前缀被复制到不同连续 buffer。

### 本文的动机

把 OS [[Virtual Memory|虚拟内存]]的抽象带到 KV cache：请求视作进程、token 视作字节、KV block 视作页。只在需要时分配页，并允许不同序列把逻辑连续块映射到非连续物理块及同一共享物理块。

---

## 方法详解

### 系统架构

vLLM 是一个面向 [[Large Language Model Serving|大语言模型服务]]的分布式推理系统：

- **输入**：prompt、最大生成长度、采样参数或 beam width；
- **调度**：中央 [[Iteration-level Scheduling|迭代级调度器]]基于 FCFS 组成 sequence group batch；
- **内存层**：[[KV Cache Manager]]维护每条序列的逻辑块、物理 GPU/CPU 块和块表；
- **执行层**：每个 worker 保存模型 shard 与相应注意力头的 KV 分片，依据共同块表执行 [[PagedAttention]]；
- **输出**：采样 token 与更新后的序列状态。

![[PagedAttention_page05.png]]

**Figure 4（原文 Fig. 4）**：中央 scheduler 向多个 model-parallel worker 发送 token 与 block table；GPU/CPU allocator 由 KV cache manager 统一控制。

### 核心模块

#### 模块 1：[[PagedAttention]]

**设计动机**：普通 self-attention 假设按位置连续读取 K/V；但逻辑 token 顺序与物理存放位置无需绑定。PagedAttention 将每 $B$ 个 token 的 K/V 打成一个 KV block，注意力内核逐块读取。

**具体实现**：
- 连续的逻辑块可以映射至离散物理块；
- attention kernel 根据块表逐块 fetch K/V，在 kernel 内完成读块与注意力计算；
- 块大小越大，读取并行度越高；越小则尾块碎片和写时复制粒度越低。论文的默认值为 16 tokens。

![[PagedAttention_page05.png]]

**Figure 5（原文 Fig. 5）**：query 依次作用于非连续 Block 0/1/2 内的 K/V；物理不连续不改变逻辑注意力窗口。

#### 模块 2：[[Block Table|逻辑-物理块映射]]

每条序列按从左到右的逻辑 KV blocks 表示。块表记录 `logical_block_id → physical_block_id` 与填充 token 数：

1. prefill 后只为 prompt 所占块分配物理块；
2. decode 时若尾块未满，只写其空 slot；
3. 尾块写满才申请一个新块；
4. 请求结束后释放其引用计数归零的块。

因此每条活跃请求最多仅浪费一个尾块，固定大小的物理块也消除了外部碎片。

![[PagedAttention_page06.png]]

**Figure 6（原文 Fig. 6）**：7-token prompt 映射到两个物理块；首轮 decode 填满尾块，下一轮才为新的逻辑块申请物理 Block 3。

![[PagedAttention_page06.png]]

**Figure 7（原文 Fig. 7）**：两个请求的相邻逻辑块可分散映射到同一 GPU block pool，不必各自拥有连续地址范围。

#### 模块 3：共享、[[Copy-on-Write|写时复制]]与多样解码

- **平行采样**：多条 sample 初始映射同一 prompt blocks；写入共享尾块时，引用计数大于 1 的序列复制一个块后再写。
- **束搜索**：候选 beam 的共同路径共享任意前缀块；淘汰候选时递减引用计数并释放归零块。它避免传统实现反复复制整个 KV cache。
- **共享前缀**：服务端预留常用 system prompt 的物理块；新请求只计算未共享的任务后缀，最后一个共享块标为 COW。
- **混合解码**：内核只接收每序列的物理 block ID 列表，因而同 batch 可混合不同采样偏好。

![[PagedAttention_page07.png]]

**Figure 8（原文 Fig. 8）**：平行采样共享 prompt 两块；首次对共享尾块写入触发块级 COW。

![[PagedAttention_page07.png]]

**Figure 9（原文 Fig. 9）**：beam 候选按分叉历史共享不同长度的块前缀，候选重排只更新映射与引用计数。

![[PagedAttention_page08.png]]

**Figure 10（原文 Fig. 10）**：机器翻译的 few-shot instruction/examples 可作为跨请求共享前缀。

#### 模块 4：[[Preemption|抢占]]与分布式执行

过载时采用 FCFS（最晚到达的先被抢占），以整个 sequence group 为 all-or-nothing 单位，因为同组序列可能共享块：

- [[KV Cache Swapping|交换]]：把整组 KV blocks 从 GPU 迁移至 CPU RAM，后续再换回；CPU swap 空间上界等于 GPU KV cache pool。
- [[KV Cache Recomputation|重计算]]：恢复时把已生成 token 与原 prompt 拼接，走一次并行 prefill 重建 KV cache。
- [[Tensor Parallelism|张量并行]]：中央管理器向全部 worker 广播相同块表；每个 worker 只存自己注意力头所需的 KV 分片，并通过 all-reduce 同步模型计算而无需同步内存管理。

---

## 关键公式

### 公式 1：[[Autoregressive Generation|自回归概率分解]]

$$
P(x_{1:n}) = P(x_1)\prod_{i=2}^{n}P(x_i\mid x_{1:i-1})
$$

**含义**：下一 token 条件于全部历史 token，因此 decode 阶段必须保留历史 K/V。

**符号说明**：
- $x_i$：第 $i$ 个 token；
- $x_{1:i-1}$：当前位置之前的上下文。

### 公式 2：[[Self-Attention|自注意力的 QKV 投影]]

$$
q_i=W_qx_i,\qquad k_i=W_kx_i,\qquad v_i=W_vx_i
$$

**含义**：每个位置生成 query、key、value；历史 $k_i,v_i$ 即构成 KV cache。

**符号说明**：
- $x_i\in\mathbb{R}^{d}$：位置 $i$ 的 hidden state；
- $W_q,W_k,W_v$：可训练线性投影；
- $q_i,k_i,v_i$：对应位置的 Q/K/V 向量。

### 公式 3：[[Causal Attention|因果缩放点积注意力]]

$$
a_{ij}=\frac{\exp(q_i^\top k_j/\sqrt d)}{\sum_{t=1}^{i}\exp(q_i^\top k_t/\sqrt d)},\qquad
o_i=\sum_{j=1}^{i}a_{ij}v_j
$$

**含义**：query $i$ 对全部历史位置归一化加权；PagedAttention 必须计算与此相同的 $o_i$。

**符号说明**：
- $d$：attention head 维度；
- $a_{ij}$：位置 $i$ 对 $j$ 的注意力权重；
- $o_i$：attention 输出。

### 公式 4：[[PagedAttention|块式注意力]]

$$
K_j=(k_{(j-1)B+1},\ldots,k_{jB}),\qquad
V_j=(v_{(j-1)B+1},\ldots,v_{jB})
$$

$$
A_{ij}=\frac{\exp(q_i^\top K_j/\sqrt d)}{\sum_{t=1}^{\lceil i/B\rceil}\exp(q_i^\top K_t/\sqrt d)},\qquad
o_i=\sum_{j=1}^{\lceil i/B\rceil}V_jA_{ij}^{\top}
$$

**含义**：将公式 3 的历史位置按块重组；块的物理地址可不连续，但对所有逻辑 token 的归一化与加权结果保持一致。

**符号说明**：
- $B$：KV block size；
- $K_j,V_j$：第 $j$ 个 key/value block；
- $A_{ij}$：query $i$ 对整个块 $j$ 的注意力分数行向量。

---

## 关键图表

> 原论文为 16 页、共 19 个 Figure、1 个 Table。由于 arXiv 未提供 HTML 图片资源，以下使用经本地渲染、可离线访问的原 PDF 页图；同页包含的全部 Figure 均逐一编号说明。

### Figure 1：GPU 内存布局与吞吐曲线

![[PagedAttention_page01.png]]

**说明**：13B 模型在 A100-40GB 上，参数约占 26GB（65%）、KV cache 超过 30%；vLLM 使 KV cache 随 batch 的增长更平缓，从而允许更大 batch。

### Figure 2：KV cache 利用率分解

![[PagedAttention_page02.png]]

**说明**：Orca(Max/Pow2/Oracle) 的真实 token state 占比依次为 20.4%、26.8%、38.2%，而 vLLM 达 96.3%。

### Figure 3：传统连续 KV cache 的三类浪费

![[PagedAttention_page04.png]]

**说明**：预留槽、内部碎片和外部碎片都阻塞其他请求进入 GPU。

### Figure 4：vLLM 系统概览

![[PagedAttention_page05.png]]

**说明**：scheduler、KV cache manager、CPU/GPU allocator 与多个 worker 的关系。

### Figure 5：PagedAttention 的非连续 KV 块读取

![[PagedAttention_page05.png]]

**说明**：同一逻辑序列的 K/V 分布在多个非连续物理块，内核逐块完成注意力。

### Figure 6：Block table 地址翻译

![[PagedAttention_page06.png]]

**说明**：逻辑块独立于物理块；末块仅部分填充时继续写入，填满后才申请下一块。

### Figure 7：并发请求的块池复用

![[PagedAttention_page06.png]]

**说明**：不同请求可在共同 block pool 中交错占据物理块。

### Figure 8：平行采样的 COW

![[PagedAttention_page07.png]]

**说明**：同一 prompt 只存一份，修改共享尾块时复制一个 block 而非整段 KV cache。

### Figure 9：束搜索的动态块共享

![[PagedAttention_page07.png]]

**说明**：候选的公共生成路径共享块；淘汰路径释放块，保留路径可再次分叉。

### Figure 10：共享系统提示词示例

![[PagedAttention_page08.png]]

**说明**：翻译 instruction 与 demonstrations 是多个请求的可缓存公共 prefix。

### Table 1：模型和服务器配置

| 模型规模 | 13B | 66B | 175B |
|---|---:|---:|---:|
| GPUs | 1× A100 | 4× A100 | 8× A100-80GB |
| 总 GPU 内存 | 40 GB | 160 GB | 640 GB |
| 参数大小 | 26 GB | 132 GB | 346 GB |
| KV cache 内存 | 12 GB | 21 GB | 264 GB |
| 最大 KV cache slots | 15.7K | 9.7K | 60.1K |

**说明**：模型增大并不意味着 KV cache 容量线性增加；66B 的参数占用导致可用 KV slots 反而少于 13B 配置。

### Figure 11：ShareGPT 与 Alpaca 的长度分布

![[PagedAttention_page09.png]]

**说明**：ShareGPT 平均输入/输出为 161.31/337.99 token，Alpaca 为 19.31/58.45；前者输入长 8.4×、输出长 5.8×，更容易暴露内存管理差异。

### Figure 12：基础单序列采样的延迟-负载曲线

![[PagedAttention_page10.png]]

**说明**：覆盖 OPT-13B/66B/175B 和两类 trace。ShareGPT 上 vLLM 在相近 normalized latency 下可持续 1.7–2.7× Orca(Oracle) 与 2.7–8× Orca(Max) 的请求率；相对 FasterTransformer 最高 22×。

### Figure 13：OPT-13B 的平均并发 batch

![[PagedAttention_page10.png]]

**说明**：ShareGPT 2 req/s 时 vLLM batch 为 30.42，Orca(Oracle) 13.62、Orca(Max) 7.00；Alpaca 30 req/s 时为 132.44、72.75、7.00。

### Figure 14：平行生成与束搜索吞吐

![[PagedAttention_page11.png]]

**说明**：在 OPT-13B/Alpaca 上，parallel size 或 beam width 越大，块共享越多，vLLM 相对 Orca 的优势越显著；beam width 6 对 Orca(Oracle) 达 2.3×。

### Figure 15：共享带来的 KV block 节省

![[PagedAttention_page11.png]]

**说明**：Alpaca 上，平行采样（2/4/6 outputs）节省 6.09%/8.53%/9.79%；束宽（2/4/6）节省 37.56%/53.13%/55.16%。ShareGPT 的对应范围为 16.2%–30.5% 与 44.3%–66.3%。

### Figure 16：共享 prefix 的翻译负载

![[PagedAttention_page12.png]]

**说明**：LLaMA-13B/WMT16 中，80-token one-shot prefix 相比 Orca(Oracle) 提升 1.67×；341-token five-shot prefix 提升 3.58×。

### Figure 17：聊天机器人负载

![[PagedAttention_page12.png]]

**说明**：截断至 1024-token prompt 的 ShareGPT 聊天负载上，vLLM 可持续约 2× Orca 的请求率；长 prompt 加剧了连续预留的浪费。

### Figure 18：内核与块大小消融

![[PagedAttention_page12.png]]

**说明**：PagedAttention attention kernel 比 FasterTransformer 高 20%–26% latency，但端到端吞吐更优；ShareGPT 的 16–128 block size 最好，短序列 Alpaca 仅 16/32 表现稳定。默认 16 是并行度与碎片的折中。

### Figure 19：重计算与交换的取舍

![[PagedAttention_page13.png]]

**说明**：小块下 swap 有大量小 PCIe 传输、带宽利用差，recompute 更优；块为 16–64 时二者端到端性能接近，且 recompute 开销不超过 swap 延迟的 20%。

---

## 实验

### 数据集与工作负载

| 数据集/工作负载 | 特征 | 用途 |
|---|---|---|
| [[ShareGPT]] | 长而高方差的真实聊天输入输出 | 基础采样、聊天、共享与块大小测试 |
| [[Alpaca]] | 较短的 instruction 数据 | 基础采样、平行采样、beam search |
| [[WMT16]] En→De | 共享 one-/few-shot translation prefix | 共享 prefix 实验 |

### 实现细节

- **模型**：OPT-13B/66B/175B，LLaMA-13B；
- **硬件**：Google Cloud A2 的 NVIDIA A100，175B 使用 8× A100-80GB；
- **并行**：Megatron-LM 风格张量并行，NCCL all-reduce；
- **实现**：8.5K 行 Python（scheduler/block manager）与 2K 行 C++/CUDA（kernels）；
- **关键 kernels**：融合 reshape+block write、融合 block read+attention、融合多 block COW copy；
- **指标**：normalized latency = 每请求端到端 latency / 输出长度；通过逐步增加 Poisson 到达率观察系统饱和点。

### 结论

高吞吐来自“更多活跃请求的 KV working set 能同时驻留”，不是单个 attention kernel 更快。PagedAttention 有 20%–26% 内核间接寻址开销，依然因显著扩大的有效 batch 而在整体服务上获胜。

---

## 批判性思考

### 优点

1. **抽象恰当且可证明地节省内存**：将 allocator 问题改造成定长块映射，直接消除外部碎片并将每请求尾部浪费限制为一个块。
2. **将内存布局与 attention kernel 协同设计**：不是仅套用虚拟内存；内核能够消费 block table，因而逻辑抽象不会破坏注意力正确性。
3. **覆盖真实服务语义**：平行采样、beam、共享前缀、抢占与模型并行都通过统一的 block table/COW 机制表达。
4. **实验说明边界条件**：短 Alpaca + OPT-175B 余量大时系统变 compute-bound，优势随之收窄，没有把收益泛化为无条件结论。

### 局限性

1. **原论文基线 Orca 为作者复现版**：Orca 不公开，尽管 Oracle/Pow2/Max 的上、下界有帮助，仍增加实现等价性的不确定性。
2. **块是静态粒度**：16-token 默认值对短请求与长请求并非同时最优；块越大，碎片、COW 和可共享边界的损失越明显。
3. **抢占语义较粗**：按 sequence group all-or-nothing 换出利于正确性与共享，但可能增大尾延迟；CPU swap 受 PCIe 小传输影响。
4. **年代与系统演进**：论文主要讨论 decode；后续长上下文、prefill/decode 解耦、量化 KV、分层缓存、prefix routing 等问题未在此设计中系统评估。

### 潜在改进方向

1. 基于请求长度、共享概率与 PCIe/NVLink 拓扑自适应选择 block size 与抢占方式。
2. 结合 KV quantization、跨节点层级缓存与 prefix-aware routing，扩展从单实例块池到集群级 cache 复用。
3. 为 shared-prefix 与动态 beam DAG 引入更细粒度的 admission control / 代价模型，平衡吞吐、TTFT 与尾延迟。

### 可复现性评估

- [x] 代码开源（[vLLM](https://github.com/vllm-project/vllm)）
- [x] 模型可获取（OPT、LLaMA 评测配置）
- [x] 内核、调度和基准配置有详细描述
- [x] 使用公开数据集/trace 长度分布

---

## 关联笔记

### 基于
- [[Transformer]]：KV cache 来自 Transformer self-attention 的历史 K/V。
- [[Virtual Memory]]：逻辑块—物理块映射和按需分配的直接灵感。
- [[Orca]]：迭代级调度与 vLLM 的内存利用优化互补。

### 对比
- [[FasterTransformer]]：内核延迟更低但缺少细粒度调度和 paged KV 管理。
- [[FlashAttention]]：优化 attention 的计算/I/O 峰值；PagedAttention 优化在线服务的持久 KV cache 布局。
- [[FlexGen]]：面向资源受限推理的 weight/token-state swapping，非在线服务中的块级共享。

### 方法相关
- [[KV Cache]]：优化对象。
- [[PagedAttention]]：核心注意力与块式存储机制。
- [[Block Table]]：逻辑-物理地址翻译层。
- [[Copy-on-Write]]：共享块发生写入时的正确性机制。
- [[Prefix Caching]]：复用公共 system prompt 的 KV states。
- [[Iteration-level Scheduling]]：允许请求在 token iteration 边界加入/退出 batch。

### 硬件/数据相关
- [[NVIDIA A100]]：主要评测 GPU。
- [[ShareGPT]]、[[Alpaca]]、[[WMT16]]：请求长度与共享 prefix 的工作负载来源。

---

## 速查卡片

> [!summary] PagedAttention（SOSP 2023）
> - **核心**: 以虚拟内存式逻辑/物理块映射管理 LLM KV cache。
> - **方法**: 非连续块注意力 + block table + 引用计数/COW + 抢占恢复。
> - **结果**: 相同延迟下对 Orca/FasterTransformer 获得 2–4× 吞吐；KV token state 占比达 96.3%。
> - **代码**: https://github.com/vllm-project/vllm

---

*笔记创建时间: 2026-07-31*
