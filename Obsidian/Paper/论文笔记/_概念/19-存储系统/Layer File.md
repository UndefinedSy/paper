---
type: concept
aliases: [层文件, Delta Layer, Image Layer, 增量层, 镜像层]
---

# Layer File

## 定义

[[Pageserver]] 中持久化层文件的两种形态，共同构成按 **page key × [[LSN]] 二维**组织的 LSM 式存储格式：

- **Delta layer（增量层）**：一段 key 范围 × 一段 LSN 范围的**原始 WAL 记录**文件。摄入时不做任何 WAL redo 或转换，因此 PS 能跟上写密集负载。
- **Image layer（镜像层）**：某个 LSN 上一段 key 范围的**整页快照**，由后台压实把 delta 记录重放到既有镜像上生成。

## 核心要点

1. **二维索引**：与只按 key 组织的经典 [[LSM-tree]] 不同，层文件同时沿 key 与 LSN 两个维度划分。
2. **读取流程**：`GetPage(key=k, lsn=l)` → ① 在 LSN < l 的层中找 k 的最新镜像 → ② 找自该镜像 LSN 到 l 之间所有触及 k 的 WAL 记录并重放。
3. **两个调参旋钮**：
   - **image threshold $I$**：某个 key range 上允许堆叠的最大 delta 层数，超过即触发镜像压实。它限定单次读的重建深度（读代价 $O(I)$）与镜像创建带来的写放大 $O(1/I)$。
   - **GC-compaction 触发比 $R$**：新累积历史达到上次压实后 base 大小的 $R$ 倍时触发，把 GC horizon 以下的历史塌缩成新 base，写放大贡献 $O(1/R)$。
4. **层数固定**：与传统 LSM 的多层结构不同，该格式**层数是常数**——作者明确不以 state-of-the-art 为目标，而是换取放大行为的**可预测性**。
5. **对象存储为真相源**：层文件持久化在[[Object Storage|对象存储]]，本地 SSD 仅缓存并可按 [[LRU]] 淘汰。
6. **分支感知**：本 timeline 无法满足的读会递归到父 timeline 的层文件（元数据在 timeline 索引文件中）。
7. **GC**：后台垃圾回收周期性移除历史数据已落在 [[PITR]] 窗口之外、且对将来重放无用的 delta 与 image 层。

## 代表工作

- [[Lakebase]]: 全部层文件的格式与调参策略，生产实测约 7× 空间放大。
- [[Neon]]: 同一存储格式的原始实现。

## 相关概念

- [[Pageserver]]
- [[LSM-tree]]
- [[Write Amplification]]
- [[PITR]]
- [[GetPage@LSN]]
- [[Timeline]]
