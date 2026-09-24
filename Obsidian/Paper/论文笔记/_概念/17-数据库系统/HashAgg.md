---
type: concept
aliases: [哈希聚合算子]
---

# HashAgg

## 定义

基于哈希表实现的聚合执行算子，将输入数据按分组键哈希分桶后完成聚合计算。

## 核心要点

1. [[TDSQL]] 的向量化实现采用 xxHash 哈希 + 线性探测，配合两级 Pointer Block–Payload Block 哈希表结构。
2. 超出内存阈值时溢出到磁盘，随后重载做归并处理。
3. 向量化执行中用 SelectionVector 把批量探测结果划分为 absent-key/existing-key/collision 三组分别处理插入、更新与继续探测，减少分支并配合 SIMD 加速哈希与键比较。

## 代表工作

- [[TDSQL]]：HashAgg 与 [[HashJoin]] 是全链路向量化重构的重点算子。

## 相关概念

- [[HashJoin]]
- [[Vectorized Execution Engine]]
- [[SIMD]]
- [[Virtual DN Parallelism]]
