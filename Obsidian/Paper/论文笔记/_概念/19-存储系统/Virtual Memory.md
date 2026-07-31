---
type: concept
aliases: [虚拟内存, Paging]
---

# Virtual Memory

## 定义

操作系统将程序的逻辑连续地址空间映射到按需分配、可以物理不连续的固定大小页的内存抽象。

## 核心要点

1. 页表解耦逻辑地址与物理地址，减少连续分配需求。
2. 固定大小页简化分配并抑制外部碎片。
3. 页可按需申请、共享、写时复制和换出。

## 代表工作

- [[PagedAttention]]：将 request/token/KV block 类比为 process/byte/page，用块表、COW 与换出管理 KV cache。

## 相关概念

- [[Block Table]]
- [[Copy-on-Write]]
- [[KV Cache Swapping]]
