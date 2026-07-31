---
title: "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness"
method_name: "FlashAttention"
authors: [Tri Dao, Daniel Y. Fu, Stefano Ermon, Atri Rudra, Christopher Ré]
year: 2022
venue: NeurIPS 2022
tags: [attention, transformer, gpu-optimization, io-aware, long-context]
zotero_collection: ""
image_source: local
arxiv_html: ""
created: 2026-07-31
---

# 论文笔记：FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness

## 元信息

| 项目 | 内容 |
|------|------|
| 机构 | Stanford University、University at Buffalo, SUNY |
| 日期 | 2022-06-24（arXiv v2） |
| 会议 | NeurIPS 2022 |
| 链接 | [arXiv](https://arxiv.org/abs/2205.14135) / [Code](https://github.com/Dao-AILab/flash-attention) |
| 对比基线 | PyTorch Attention、Megatron-LM、Apex FMHA、Linformer、Performer、Reformer、Longformer |

---

## 一句话总结

> FlashAttention 不是近似注意力：它以 [[Tiling|SRAM 分块]]、[[Online Softmax|在线 softmax]] 与反向重算避免把 $N\times N$ attention 矩阵写入 [[High Bandwidth Memory|HBM]]，在保持精确结果的同时，把训练注意力从内存受限操作变为更高效的 IO-aware kernel。

---

## 核心贡献

1. **把数据移动而非 FLOPs 作为优化对象**：指出标准 attention 多次物化 $S=QK^\top$ 与 $P=\operatorname{softmax}(S)$，造成 $\Theta(Nd+N^2)$ HBM 访问；计算量降低不必然带来 wall-clock 加速。
2. **精确的 IO-aware attention**：通过 $Q/K/V$ tile、[[Online Softmax|可组合的数值稳定 softmax 统计量]]与 [[Kernel Fusion|单 CUDA kernel 融合]]，不在 HBM 保存 $S,P$，额外空间从 $O(N^2)$ 降到 $O(N)$。
3. **重算换取更少 IO**：前向只保存输出 $O$ 及每行 $(m,\ell)$；反向在 SRAM 重算局部 $S,P$，虽增加 FLOPs，却减少大矩阵 HBM 读写并加速 backward。
4. **IO 分析与扩展**：证明 dense FlashAttention 的 HBM complexity 为 $\Theta(N^2d^2/M)$，并给出不能对所有 SRAM 大小渐近优于它的下界；同一原语扩展为 [[Block-Sparse Attention|block-sparse]] FlashAttention。

---

## 问题背景

### 标准 [[Self-Attention|自注意力]]的瓶颈

给定 $Q,K,V\in\mathbb{R}^{N\times d}$，标准实现依次计算：

$$
S=QK^\top\in\mathbb{R}^{N\times N},\qquad
P=\operatorname{softmax}(S),\qquad
O=PV\in\mathbb{R}^{N\times d}.
$$

即使矩阵乘法本身具备高算术强度，常规训练实现仍会：写 $S$ 到 HBM、读 $S$ 写 $P$、读 $P,V$ 写 $O$；mask/dropout 还会加入额外 kernel 与 HBM 往返。因此长序列时，瓶颈是 [[Memory-Bound Operation|内存访问]]，不是数学上的 $O(N^2d)$ FLOPs。

### 硬件动机

以 A100 为例，HBM 容量约 40–80 GB、带宽约 1.5–2.0 TB/s；每个 SM 的片上 SRAM 仅约 192 KB，却有约 19 TB/s 带宽。FlashAttention 的关键选择是：**把一个 attention tile 留在 SRAM/register 内完成 matmul、mask、softmax、dropout 与第二次 matmul，而非把中间矩阵存回 HBM。**

---

## 方法详解

### 算法 0：标准 attention 的 IO 问题

1. 从 HBM 分块读 $Q,K$，计算 $S=QK^\top$，把 $S$ 写回 HBM；
2. 读 $S$，计算 $P=\operatorname{softmax}(S)$，把 $P$ 写回 HBM；
3. 读 $P,V$，计算 $O=PV$，写回 $O$。

大矩阵 $S,P$ 的物化同时造成二次内存占用和高 HBM 访问量。

### 算法 1：dense FlashAttention

设 SRAM 大小为 $M$，head dimension 为 $d$，取：

$$
B_c=\left\lceil\frac{M}{4d}\right\rceil,
\qquad
B_r=\min\left(\left\lceil\frac{M}{4d}\right\rceil,d\right).
$$

算法以 $K_j,V_j$ 为外层 block、以 $Q_i$ 为内层 block：

1. 将一个 $K_j,V_j$ tile 从 HBM 放入 SRAM；
2. 依次读取 $Q_i,O_i,\ell_i,m_i$；
3. 在 SRAM 中算 $S_{ij}=Q_iK_j^\top$，并得到 tile 的 row max $\tilde m_{ij}$ 与未归一化指数和 $\tilde\ell_{ij}$；
4. 用新旧统计量重标定已有输出并累加 $\tilde P_{ij}V_j$；
5. 只写回更新后的 $O_i,\ell_i,m_i$。

因此 $S_{ij}$ 与 $\tilde P_{ij}$ 都是瞬时 tile，不会成为 HBM 中的 $N\times N$ 张量。

### [[Online Softmax|在线 softmax]]：精确分块归一化

对向量 $x$ 定义：

$$
m(x)=\max_i x_i,
\qquad
f(x)_i=e^{x_i-m(x)},
\qquad
\ell(x)=\sum_i f(x)_i,
\qquad
\operatorname{softmax}(x)=\frac{f(x)}{\ell(x)}.
$$

把两段 $x^{(1)},x^{(2)}$ 拼接时：

$$
m=\max(m_1,m_2),
\qquad
\ell=e^{m_1-m}\ell_1+e^{m_2-m}\ell_2.
$$

每行只需保存 $(m,\ell)$，就能把任意后续 $K/V$ block 合并为与一次性 softmax 完全相同的结果；这是算法能够同时“分块”和“精确”的关键。

### 输出更新与反向 [[Recomputation|重计算]]

处理 tile 后，算法用新的归一化统计量更新输出：

$$
O_i\leftarrow \operatorname{diag}(\ell_i^{new})^{-1}
\left[
\operatorname{diag}(\ell_i)e^{m_i-m_i^{new}}O_i+
 e^{\tilde m_{ij}-m_i^{new}}\tilde P_{ij}V_j
\right].
$$

训练时不保存 $S,P$；前向保存 $O,m,\ell$（dropout 时另保存随机数状态）。反向再按相同 tile 重算 $S_{ij}$ 和：

$$
P_{ij}=\operatorname{diag}(\ell_i)^{-1}
\exp(S^{masked}_{ij}-m_i),
\qquad
D_i=\operatorname{rowsum}(dO_i\odot O_i),
$$

并在 SRAM 更新 $dQ,dK,dV$。这是选择性 [[Gradient Checkpointing|梯度检查点]]：用廉价的局部计算替换昂贵的二次 HBM 中间状态读取。

### [[Kernel Fusion|内核融合]]

一个 CUDA kernel 完成：

```text
QKᵀ → mask → stable softmax → dropout（可选）→ PV
```

融合本身减少 launch 与逐算子 HBM 往返；更重要的是，与 tiling 配合使中间 tile 根本不离开片上存储。论文也强调：只做普通 fusion、但为 backward 保存整个 attention matrix，仍无法解决长序列内存问题。

### IO complexity 与最优性

- 标准 attention：$\Theta(Nd+N^2)$ 次 HBM 访问；
- FlashAttention：$\Theta(N^2d^2/M)$；
- 额外内存：$O(N)$；
- 算术量：保持 $O(N^2d)$，因此它不是将 dense exact attention 降为线性复杂度的算法。

当 $d=64\sim128$、$M\approx100$ KB 时，$d^2\ll M$，所以 tile 复用显著降低 HBM 访问。论文的下界说明：对 $d\le M\le Nd$，不存在能在**所有** $M$ 上以 $o(N^2d^2/M)$ HBM accesses 完成 exact attention 的算法。

### [[Block-Sparse Attention|Block-sparse FlashAttention]]

给定块掩码，只对非零 $(Q_i,K_j)$ tiles 执行 dense tile 更新，零块直接跳过。若非零块比例为 $s$：

$$
\Theta\left(Nd+\frac{N^2d^2}{M}s\right)
$$

当 $s=N^{-1/2}$ 或 $s=1/\log N$ 时，可分别达到 $\Theta(N\sqrt N)$ 或 $\Theta(N\log N)$ IO。实验采用 fixed butterfly sparsity；它牺牲 dense exact attention 语义，但能结合稀疏性和 IO locality。

---

## 关键图表与表格

> 原论文共 34 页、8 张 Figure、21 张 Table、算法 0–5。arXiv HTML 不可用，以下嵌入本地渲染的原 PDF 页面；每张图与每张表均单独说明。

### Figure 1：内存层级、tile 执行顺序与端到端动机

![[FlashAttention_page02.png]]

左侧显示 SRAM/HBM/DRAM 的带宽与容量鸿沟，中部给出 $K,V$ 外循环、$Q$ 内循环的分块顺序，右侧报告 GPT-2 attention 相对 PyTorch 最高 **7.6×** 加速。

### Figure 2：HBM 访问量主导时间，block size 存在饱和点

![[FlashAttention_page06.png]]

GPT-2 Medium（$N=1024,d=64$，A100）中，标准实现 66.6 GFLOPs、40.3 GB HBM R/W、41.7 ms；FlashAttention 反而有 75.2 GFLOPs，却只需 4.4 GB、7.3 ms。增大 block size 会减少 HBM passes，直到约 256 后逐渐转为算力或 SRAM 容量受限。

### Figure 3：runtime/memory 随序列长度的整体对比

![[FlashAttention_page09.png]]

dense FlashAttention 的运行时间仍为二次增长，但在常见长度上快于多种 approximate/sparse baseline；显存则线性增长，最多比 exact baseline 少约 20×。block-sparse 版本在更长序列上保持近线性优势。

### Figure 4：数值正确性

![[FlashAttention_page27.png]]

GPT-2 small/medium 的 validation perplexity 曲线与 HuggingFace 基线几乎重合，验证在线归一化与重算没有改变训练结果。

### Figure 5：A100、$d=64$ 的加速曲线

![[FlashAttention_page28.png]]

对不同 sequence length，常见配置普遍获得约 2–4×；使用 mask/dropout 时，由于融合避免更多中间 IO，优势更明显。

### Figure 6：A100、$d=128$ 的加速曲线

![[FlashAttention_page29.png]]

head dimension 增大使 tile 占用更多 SRAM、block size 变小，收益下降；但 causal mask 使一半 blocks 可跳过，仍可达约 3×。

### Figure 7：RTX 3090 的加速曲线

![[FlashAttention_page29.png]]

RTX 3090 的 HBM 带宽低于 A100（约 900 GB/s vs. 1.5 TB/s），所以 memory-bound baseline 更弱，论文测得约 2.5–4.5× speedup。

### Figure 8：T4 上的训练/推理加速

![[FlashAttention_page30.png]]

T4 的 SRAM 更小，必须使用更小 tile，收益相对较低；论文同时给出 forward+backward 与仅 forward，用于说明推理场景。

### Table 1：BERT-large 达标训练时间

| 实现 | 达到 MLM 72.0% 的时间（8×A100） |
|---|---:|
| Nvidia MLPerf 1.1 | $20.0\pm1.5$ min |
| FlashAttention | $17.4\pm1.4$ min |

相对当时 MLPerf 参考实现快约 **15%**。

### Table 2：GPT-2 端到端训练

| 模型 | 实现 | PPL | 时间 | 相对 HuggingFace |
|---|---|---:|---:|---:|
| GPT-2 Small | HuggingFace | 18.2 | 9.5 d | 1.0× |
| GPT-2 Small | Megatron-LM | 18.2 | 4.7 d | 2.0× |
| GPT-2 Small | FlashAttention | 18.2 | 2.7 d | 3.5× |
| GPT-2 Medium | HuggingFace | 14.2 | 21.0 d | 1.0× |
| GPT-2 Medium | Megatron-LM | 14.3 | 11.5 d | 1.8× |
| GPT-2 Medium | FlashAttention | 14.3 | 6.9 d | 3.0× |

### Table 3：Long Range Arena

| 方法 | Avg accuracy | Speedup |
|---|---:|---:|
| Transformer | 59.3 | – |
| FlashAttention | 59.8 | 2.4× |
| Block-sparse FlashAttention | 59.6 | 2.8× |
| Linformer / Linear / Performer | 54.9 / 59.6 / 58.9 | 2.5× / 2.3× / 1.8× |
| Local / Reformer / SMYRF | 56.0 / 57.6 / 57.9 | 1.7× / 1.3× / 1.7× |

dense exact 版本维持标准 Transformer 精度；block-sparse 在这里也没有明显精度损失，并快于所测近似方法。

### Table 4：更长 GPT-2 context 的收益

| 实现 | Context | PPL | 训练时间 |
|---|---:|---:|---:|
| Megatron-LM | 1K | 18.2 | 4.7 d |
| FlashAttention | 1K | 18.2 | 2.7 d |
| FlashAttention | 2K | 17.6 | 3.0 d |
| FlashAttention | 4K | 17.5 | 3.6 d |

4K FlashAttention 比 1K Megatron 仍快约 30%，PPL 改善 0.7。

### Table 5：长文档分类（micro-$F_1$）

| 数据集 | 512 | 1K | 2K | 4K | 8K | 16K |
|---|---:|---:|---:|---:|---:|---:|
| MIMIC-III | 52.8 | 50.7 | 51.7 | 54.6 | 56.4 | 57.1 |
| ECtHR | 72.2 | 74.3 | 77.1 | 78.6 | 80.7 | 79.2 |

更长 context 在 MIMIC-III 取得 +4.3、在 ECtHR 取得 +8.5（相对 512 的最佳长度）。

### Table 6：Path-X / Path-256

| 方法 | Path-X | Path-256 |
|---|---:|---:|
| 既有 Transformer、Linformer、Linear、Performer、Local、Reformer、SMYRF | 7.0 | 7.0 |
| FlashAttention | **61.4** | 7.0 |
| Block-sparse FlashAttention | 56.0 | **63.1** |

FlashAttention 使 Transformer 首次在 16K Path-X 超越随机；block-sparse 进一步支持 64K Path-256。

### Table 7：与 Apex FMHA 的短序列比较

![[FlashAttention_page28.png]]

A100、BERT-large 尺寸、含 mask/dropout 下：FlashAttention 的 forward 通常略快，但 backward 因重算略慢；合并后在 $N=128/256/512$ 分别约 4% 慢、8% 快、5% 快。关键差异不是短序列绝对速度，而是 FMHA 保存 $P$、只支持 $d=64,N\le512$，FlashAttention 支持至 64K 和更低 memory footprint。

### Table 8：附录 benchmark 配置索引

![[FlashAttention_page31.png]]

以 dropout/masking 的四种组合与 forward/backward/combined 三种 pass，索引到 Tables 9–21；避免只挑选一种 kernel 配置报告结果。

### Tables 9–11：同时启用 dropout 与 masking

![[FlashAttention_page31.png]]
![[FlashAttention_page32.png]]

- **Table 9（forward）**：FlashAttention 从 $N=128$ 的 0.04 ms 扩展至 $N=64$K 的 2682.35 ms；block-sparse 仅 13.34 ms。
- **Table 10（backward）**：FlashAttention $N=64$K 为 6660.44 ms；block-sparse 为 47.61 ms。
- **Table 11（combined）**：FlashAttention $N=1$K 为 2.55 ms，对 PyTorch 的 8.29 ms；$N=64$K 为 9341.30 ms，block-sparse 为 64.11 ms。

### Tables 12–14：仅 masking

![[FlashAttention_page32.png]]
![[FlashAttention_page33.png]]

- **Table 12（forward）**：FlashAttention 在 $N=1$K/4K 为 0.68/8.40 ms；block-sparse 为 0.11/0.68 ms。
- **Table 13（backward）**：FlashAttention 在 $N=1$K/4K 为 1.84/22.25 ms；block-sparse 为 0.57/3.13 ms。
- **Table 14（combined）**：FlashAttention 在 $N=1$K/4K 为 2.37/30.77 ms；block-sparse 为 0.69/3.89 ms。

### Tables 15–17：仅 dropout

![[FlashAttention_page33.png]]

- **Table 15（forward）**：FlashAttention 在 $N=1$K/4K 为 0.83/10.38 ms；block-sparse 为 0.61/1.10 ms。
- **Table 16（backward）**：FlashAttention 在 $N=1$K/4K 为 1.86/26.21 ms；block-sparse 为 0.40/2.04 ms。
- **Table 17（combined）**：FlashAttention 在 $N=1$K/4K 为 2.52/36.70 ms；block-sparse 为 0.92/3.50 ms。

### Tables 18–20：无 dropout、无 masking

![[FlashAttention_page34.png]]

- **Table 18（forward）**：FlashAttention 在 $N=1$K/4K 为 0.68/8.42 ms；block-sparse 为 0.65/0.96 ms。
- **Table 19（backward）**：FlashAttention 在 $N=1$K/4K 为 1.62/21.57 ms；block-sparse 为 0.38/2.34 ms。
- **Table 20（combined）**：FlashAttention 在 $N=1$K/4K 为 2.29/30.09 ms；block-sparse 为 0.88/3.21 ms。

### Table 21：全量 attention memory usage

![[FlashAttention_page34.png]]

| 序列长度 | PyTorch | Linformer | FlashAttention | Block-sparse FlashAttention |
|---|---:|---:|---:|---:|
| 1K | 1184 MB | 287 MB | **209 MB** | **209 MB** |
| 4K | 17024 MB | 1652 MB | **836 MB** | **836 MB** |
| 64K | OOM | 26252 MB | **13376 MB** | **13384 MB** |

两种 FlashAttention 的 footprint 都随 $N$ 线性增长；dense exact variant 在 64K 仍能运行，而多数基线已 OOM。

---

## 实验设置与结论

- **BERT**：8×A100-80GB、FP16、LAMB、batch 448；10 次重复平均。
- **GPT-2**：8×A100-40GB、OpenWebText、400K steps、mixed precision；仅替换 attention 实现，验证 PPL 曲线重合。
- **LRA**：各 task 长度 1K–4K；在相近/更高 accuracy 下报告几何平均 wall-clock speedup。
- **kernel microbenchmark**：单 A100-40GB，8 heads、$d=64$、batch 16；测量 100 次均值，比较 exact / approximate / sparse 实现。

结论应分层理解：FlashAttention **没有消除 dense attention 的二次算术复杂度**；它首先让 exact attention 不再因中间状态而二次占内存，并在中短序列获得更好的实测时间。若序列足够长且可接受近似，结构化 sparse 的 block-sparse 变体才进一步改变随 $N$ 的增长趋势。

---

## 批判性思考

### 优点

1. **目标函数正确**：论文用 HBM IO 解释了“线性/近线性近似算法不一定更快”的现实反例，把算法复杂度与硬件执行模型对齐。
2. **精确性与工程性并存**：online softmax 保障结果等价，CUDA fusion 与 tile schedule 将理论 IO 目标落实为可训练 kernel。
3. **重算不是盲目牺牲**：它针对 memory-bound 区间，用额外算术换取更少慢速存储访问，结果同时降低显存和时间。
4. **评测覆盖广**：包含端到端 BERT/GPT-2、长上下文质量、LRA、不同 GPU/head dimension，以及完整的 dropout/masking benchmark 表。

### 局限性

1. **论文的“最优”是特定 IO 模型中的渐近结论**：单 GPU、HBM/SRAM 两层与特定 $M$ 范围，不直接等价于多 GPU、NVLink、缓存层或现代硬件上的全局最优。
2. **仍为 $O(N^2d)$ dense compute**：极长序列下即便不 OOM，runtime 仍二次增长；block-sparse 需要选择 sparsity pattern，并不等价于原始 dense attention。
3. **低层实现门槛高**：每种新 attention 变体都需要 CUDA 专门实现；论文自身把高层 DSL/编译到 IO-aware kernel 列为未来工作。
4. **版本与硬件时效性**：基准以 A100/Turing/Ampere 和 2022 生态为主；后续 FlashAttention-2/3、PyTorch SDPA 与新 GPU 的 kernel 策略需单独评估。

### 与 [[PagedAttention]] 的关系

两者都优化 attention 相关内存，却在不同时间尺度工作：

- **FlashAttention**：训练或 prefill 中一次 attention 调用的临时 $S/P$，优化片上 SRAM 与 HBM 间的数据流；
- **PagedAttention**：在线 decode 中跨 token 持续存在的 [[KV Cache]]，优化逻辑/物理块布局、共享与服务调度；
- 可组合：前者减少 attention kernel 的峰值临时显存/IO，后者提高持久 KV cache 的容量与复用率。

---

## 关联笔记

### 基于
- [[Self-Attention]]：精确计算的对象。
- [[Tiling]]、[[Online Softmax]]：允许逐 tile 精确聚合。
- [[High Bandwidth Memory]]、[[SRAM]]：论文的关键硬件层级。

### 对比
- [[PagedAttention]]：后者针对在线 serving 的持久 KV cache，而非本文训练时的瞬时 attention matrix。
- [[Gradient Checkpointing]]：同样以重算节省存储，但 FlashAttention 明确以 HBM IO 为速度目标。
- [[Block-Sparse Attention]]：通过改变连接模式继续缩减长序列成本。

---

## 速查卡片

> [!summary] FlashAttention（NeurIPS 2022）
> - **核心**：以 SRAM tiling + online softmax + recomputation 精确计算 attention，不物化 $N\times N$ 的 $S,P$。
> - **复杂度**：计算仍为 $O(N^2d)$；额外内存 $O(N)$；HBM IO 为 $\Theta(N^2d^2/M)$。
> - **结果**：GPT-2 attention 最高 7.6×；BERT-large 15% 更快；GPT-2 最高 3.5×；64K dense attention 仍可运行。
> - **代码**：https://github.com/Dao-AILab/flash-attention

---

*笔记创建时间: 2026-07-31*
