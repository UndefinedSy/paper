---
type: concept
aliases: [GetPage, GetPageAtLSN]
---

# GetPage@LSN

## 定义

[[Pageserver]] 暴露给计算节点的**唯一读 API**：给定页号与 [[LSN]]，返回该页在该 LSN 上的版本。它没有 range-scan 原语，因此所有扫描型访问必须依赖计算侧缓存或外部分析引擎。

## 核心要点

1. **解析过程**：① 在 LSN 小于目标值的层中找该 key 的最新镜像（来自 image layer）→ ② 收集自该镜像 LSN 到目标 LSN 之间所有触及该 key 的 delta [[WAL]] 记录 → ③ 重放这些记录 → 返回页。
2. **代价**：$O(I)$ 次层访问，$I$ 为 image threshold，它限定了必须重放的 delta 链长度。
3. **阻塞语义**：若 PS 尚未摄入到请求所需的水位，请求会**阻塞等待**。
4. **双 LSN 优化**：请求携带 request LSN（页版本）与 not-modified-since LSN（承诺页在两者之间未变）。若只按 current LSN 请求，PS 必须一直等到追平该 LSN（因为页可能在其 last record LSN 与 current LSN 之间被改过）；带上前者后 PS 只需摄入到较低水位，显著降低延迟。not-modified-since 的值来自计算侧的 LwLSN 缓存（命中取条目，未命中取全局 last-written LSN）。
5. **未命中路径**：计算侧本地文件缓存未命中时才发 GetPage；PS 可能还需从[[Object Storage|对象存储]]拉取层文件。返回的页写回计算侧缓存并更新 LwLSN。预取读走同一路径。
6. **启动场景**：计算启动时改为请求 `GetBaseBackup`（启动所需元数据文件的 tarball），从而**避免 checkpoint**，恢复点取 PS 的 last record LSN 而非最后一个 checkpoint。

## 代表工作

- [[Lakebase]]: 计算侧缓存命中率通常 > 98.5%，掩盖了大部分 GetPage 延迟。
- [[Neon]]: 同一 API 的设计源头。

## 相关概念

- [[Pageserver]]
- [[Layer File]]
- [[LSN]]
- [[Local File Cache]]
- [[LwLSN Cache]]
