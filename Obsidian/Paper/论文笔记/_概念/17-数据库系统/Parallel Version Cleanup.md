---
type: concept
aliases: [并行清理, 动态并行清理, Dynamic Parallel Cleanup]
---

# Parallel Version Cleanup

## 定义

SQL Azure 版本清理器的自适应并行化机制：仅当系统资源充足且某库的 [[Persistent Version Store|PVS]] 体量超过阈值（生产中为 100GB）时才在库内/跨库启用多线程清理，体量降回阈值以下则恢复单线程默认路径。

## 核心要点

1. 并行化同时应用于跨数据库（Elastic Pool 场景）和单库内部（大表/大 PVS 场景）两个层面。
2. 设有 CPU 使用上限，避免抢占用户工作负载资源；只在有需要且资源允许时才引入并行开销，保护小体量实例不被清理拖累。
3. 生产验证：满 sweep 平均时长从数天降到数小时；在 PVS≥100GB 的数据库上，P99.99 sweep 时长从约 6 天（515,121s）降到约 1 天（79,245s）。
4. 需要与 [[Read-Ahead]]、版本链短路指针配合，共同解决单线程清理在 Hyperscale（单库可超 100TB）等超大规模场景下的扩展性瓶颈。

## 代表工作

- [[AzureMVCCVersionCleaner]]：提出者，生产 30 天遥测量化其对 sweep 尾延迟的改善。

## 相关概念

- [[Elastic Pool]]
- [[Read-Ahead]]
- [[Persistent Version Store]]
