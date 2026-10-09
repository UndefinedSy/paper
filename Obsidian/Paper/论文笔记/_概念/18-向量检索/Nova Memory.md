---
type: concept
aliases: [NovaM, Nova Memory]
---

# Nova Memory

## 定义

[[Nova]] 中面向内存驻留场景的图索引变体，以 [[HNSW]] 为核心算法，并在工程上支持重复向量合并、空间复用与无锁 `scan`/`insert`/`vacuum`。

## 核心要点

1. **索引布局**：元数据（$N_{vec}$、$N_{del}$、$L_{max}$、入口节点 $v_0$）+ 多个固定容量 base file（可按需动态增加以扩容）；向量与图结构**解耦存储**，node offset 记录节点位置。
2. **空闲节点数组**：管理已删除节点的空间复用——插入时优先从数组尾部取空闲槽，并**复用该节点原有层数**而非重新随机生成。
3. **重复向量合并**：允许**一个节点关联多个 `ctid`**。重复来源有两个——应用显式插入，以及 [[MVCC]] 下非 HOT 更新传播产生的相同向量新索引项。插入时若 layer 0 有候选邻居在距离阈值内且 `ctid` 列表未满，就无锁追加。
4. **三态节点与三阶段 vacuum**：节点分 active / deleted / free；vacuum 依次执行 `remove_heap_ctids`、`repair_graph`、`mark_free`。
5. **无锁设计（三个并发场景）**：`remove_heap_ctids` vs scan 用原地双指针 + 原子操作（临时重复由合并去重消解）；`repair_node` 生成全新邻居列表并逐个原子替换；`mark_free` 只把槽位加入空闲数组、不清空内容，并配内存屏障实现 release-acquire 语义。
6. **两个上层机制**：支持 [[Iterative Query|迭代查询]]（增量扩张搜索前沿）与 [[Covering Index|覆盖索引]]（把标量属性物化进图节点）。

## 代表工作

- [[Nova]]: 作为内存场景的索引变体；实验显示 GIST-1M 上 95.7% recall 时达 6000+ QPS、P99 4.5ms。

## 相关概念

- [[HNSW]]
- [[Nova Disk]]
- [[Delta File]]
- [[Iterative Query]]
- [[Covering Index]]
- [[MVCC]]
