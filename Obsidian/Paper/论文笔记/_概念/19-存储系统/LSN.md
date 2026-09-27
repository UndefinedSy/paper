---
type: concept
aliases: [Log Sequence Number, 日志序列号]
---

# LSN

## 定义

[[WAL]] 字节流中的**字节偏移量**，是数据库日志的全局单调坐标。在 [[Neon]] / [[Lakebase]] 中它被提升为贯穿计算、[[Safekeeper]]、[[Pageserver]] 三层的一致性锚点：任何时刻的数据库状态都可由 LSN 唯一定位。

## 核心要点

1. **计算侧四条**：① current LSN（WAL 追加点）② commit LSN（quorum 持久点，崩溃恢复恢复到此处）③ replica replay LSN（副本可见点）④ static replica LSN（被钉住的读点，用于时间旅行查询）。
2. **SK 侧三条**：⑤ flush LSN ⑥ commit LSN（SK 只将其下的数据流给 PS）⑦ backup LSN（已上传对象存储的水位，其下本地段可删）。
3. **PS 侧四条**：⑧ last record LSN（内存摄入点，也是计算启动时的 redo 点）⑨ flush LSN（批量落盘点，崩溃从此再水化）⑩ remote consistent LSN（已上传远端）⑪ branch LSN（分支分叉点）。
4. **持久性判定**：一条记录在 SK quorum 刷盘后 commit LSN 才推进——这是"提交"与"持久"的统一判据。
5. **branch LSN 约束**：必须保证分支创建那一刻父分支上该 LSN 处的**每一个页都可被重建**；实现上简化为限制在父分支的 [[PITR]] 窗口内。
6. **双 LSN 读请求**：`GetPage@LSN` 携带 request LSN（要取的页版本）与 not-modified-since LSN（承诺该页在两者之间未变）——后者让 PS 只需摄入到较低水位，避免为等待 current LSN 而阻塞。not-modified-since 值由计算侧的 LwLSN 缓存提供。

## 代表工作

- [[Lakebase]]: LSN 体系是该论文 Figure 4 的全部内容，也是分支、时间旅行、副本可见性、GC 边界的统一抽象。
- [[PostgreSQL]]: LSN 概念的来源。

## 相关概念

- [[WAL]]
- [[Pageserver]]
- [[Safekeeper]]
- [[PITR]]
- [[Database Branching]]
- [[GetPage@LSN]]
