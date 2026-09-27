---
type: concept
aliases: [LwLSN, last-written-LSN cache]
---

# LwLSN Cache

## 定义

计算节点上的**固定大小**映射表，把每个页号映射到其 **not-modified-since 的 [[LSN]]**（最后一次写入该页的 LSN）。它服务于 `GetPage@LSN` 的双 LSN 请求优化。

## 核心要点

1. **解决问题**：直接按 current LSN 请求页会很慢——[[Pageserver]] 必须一直阻塞到摄入追平 current LSN，因为页可能在 PS 的 last record LSN 与 current LSN 之间被修改过。
2. **机制**：请求携带 not-modified-since LSN，承诺该页在此 LSN 到请求 LSN 之间未被修改，因此 PS 只需摄入到**较低**的 not-modified-since 水位。
3. **取值规则**：页在 LwLSN 中命中则用条目值；未命中则退化为全局 last-written LSN。
4. **更新时机**：当**脏页被淘汰**时，把该页的 LSN 写入 LwLSN 条目。
5. **配套写入**：GetPage 返回的页镜像被写入 [[Local File Cache]] 后，同时更新 LwLSN 缓存。

## 代表工作

- [[Lakebase]]: 读路径上的关键小设计，与 LFC 共同构成计算侧读优化。
- [[Neon]]: 同一机制。

## 相关概念

- [[GetPage@LSN]]
- [[Local File Cache]]
- [[LSN]]
