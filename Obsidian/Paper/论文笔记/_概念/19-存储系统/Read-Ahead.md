---
type: concept
aliases: [读预取, Readahead]
---

# Read-Ahead

## 定义

版本清理器在扫描 [[Page Free Space (PFS)|PFS]] 页定位含版本数据页时，先批量收集所有版本位被置位的页，再以轮询（round robin）方式跨文件统一发起预取 I/O，而非逐页同步读取。

## 核心要点

1. 目的是减少 I/O 等待延迟：预取把页提前载入内存，清理时避免为每个页单独承受磁盘/远程存储延迟。
2. 轮询跨文件预取可避免单一文件被预取请求集中打满。
3. 是 SQL Azure 版本清理可扩展性改造中最基础的一项优化，与动态并行清理、版本链短路指针共同解决单线程清理慢的问题。

## 代表工作

- [[AzureMVCCVersionCleaner]]：应用于版本清理器扫描含版本页的阶段。

## 相关概念

- [[Persistent Version Store]]
- [[MVCC]]
