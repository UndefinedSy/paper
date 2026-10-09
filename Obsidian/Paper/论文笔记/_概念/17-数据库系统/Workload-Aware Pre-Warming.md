---
type: concept
aliases: [workload-aware pre-warming, 工作负载感知的索引预热, 索引预热]
---

# Workload-Aware Pre-Warming

## 定义

根据预测的访问热度、人工指定与资源代价综合排序，主动把索引加载进内存以消除冷启动磁盘 I/O 延迟的机制。

## 数学形式

$$
P_i = \alpha \cdot S_{\text{pred}} + \beta \cdot S_{\text{man}} - \gamma \cdot S_{\text{cost}}
$$

## 核心要点

1. **预测分 $S_{\text{pred}}$**：来自 [[ARIMA]] 时序模型的置信度，分析索引历史 QPS 以预测未来访问高峰。
2. **人工分 $S_{\text{man}}$**：反映人工提交的请求，并按请求来源的历史准确率（声誉）加权，保证可靠性。
3. **代价分 $S_{\text{cost}}$**：与索引磁盘体积成正比，用于抑制大索引优先级以促进公平分配。
4. **超参 $\alpha, \beta, \gamma$**：分别平衡预测趋势、人工输入与资源代价。
5. **执行方式**：调度器按优先队列把索引的 `mmap` 区域加载进物理内存（受系统资源约束）——把手工预热变成**自动化的性能保证**。

## 代表工作

- [[Nova]]: 用于多租户场景的冷启动优化，与 [[Anytime Query]] 共同改善首次查询稳定性。

## 相关概念

- [[Cold Start]]
- [[Anytime Query]]
- [[ARIMA]]
- [[Noisy Neighbor]]
