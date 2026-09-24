---
type: concept
aliases: [Bloom过滤器, 布隆过滤器]
---

# Bloom Filter

## 定义

一种空间高效的概率数据结构，用于判断元素是否在集合中。可能有假阳性（判断在但实际不在），但无假阴性（判断不在则一定不在）。

## 数学形式

使用 $k$ 个独立哈希函数将元素映射到位数组 $m$ 的 $k$ 个位置，查询时检查所有位置是否为 1。

假阳性概率：$p \approx (1 - e^{-kn/m})^k$，其中 $n$ 为元素数量。

## 核心要点

1. 空间效率极高，比哈希表节省一个数量级以上的内存
2. 无假阴性保证：判断不在则一定不在
3. 在 [[Bigtable]] 中用于减少读路径的 SSTable 搜索范围
4. **分片 Bloom Filter**: 为 key range 子集维护独立 filter，DRAM 占用降低 10x
5. **混合 Bloom Filter**: 支持 column family only 查询，利用率提升 4x
6. **自适应性**: 定期评估有效性，无效时自动关闭

## 代表工作

- [[Bigtable]]: 读路径优化，减少不必要的磁盘访问
- [[Cassandra]]: SSTable 查找优化
- [[TDSQL]]: 作为三类 [[Runtime Filter]]（Bloom Filter、Range Filter、In Filter）之一，用于 Join 场景的下推剪枝

## 相关概念

- [[LSM-tree]]
- [[SSTable]]
- [[Bigtable]]
