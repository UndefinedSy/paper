---
type: concept
aliases: [DTS, 直写存储, Direct-to-Storage Ingestion]
---

# Direct-to-Storage (DTS)

## 定义

[[Lakebase]] 中绕过 PostgreSQL 计算层、由分布式 Spark 执行器**并行构造 PostgreSQL 兼容页文件并直接写入[[Object Storage|对象存储]]**的批量摄入模式。它是"开放存储 + 多引擎写"的具体落地形态。

## 核心要点

1. **解决的瓶颈**：常规 `COPY` 路径把摄入**串行化到单个 PostgreSQL writer** 上；DTS 消除该瓶颈，获得水平扩展能力。
2. **实测加速**：写 1-KB 行（混合文本与整数列）的表，相对 COPY：5 GB 2.6× / 10 GB 3.4× / 50 GB 6.2× / 100 GB 7.9× / 500 GB 31.3× / **1 TB 73.3×**，随规模单调增长。
3. **原子性在关系粒度**：Spark 先物化完整表页并全部上传，随后计算在**一个事务**里做**仅元数据的关系交换**（metadata-only relation swap）。
4. **隔离性**：新页用**冻结可见性元数据**生成，使它们对被交换快照可见的事务立即可见。
5. **持久性**：提交前所有页已落 S3，关系交换的 [[WAL]] 记录再走标准 [[Safekeeper]] 复制协议。
6. **一致性代价**：对正在做 DTS 快照装载的表，**并发 DML 会被拒绝**——这是目前"开放写入"的边界。
7. **适用场景**：数据集初次加载、时间分区数据集的周期性同步、批式 ML 特征计算（可与实时数据合并形成长窗口特征）；论文展望未来可推广到分布式索引构建与物理 schema 变更。

## 代表工作

- [[Lakebase]]: Synced Tables 的两条写入路径之一（另一条是 Spark 流式读 Delta change feed 更新 PostgreSQL 表）。

## 相关概念

- [[Lakebase]]
- [[Object Storage]]
- [[Lakehouse]]
- [[Unity Catalog]]
- [[Apache Arrow]]
