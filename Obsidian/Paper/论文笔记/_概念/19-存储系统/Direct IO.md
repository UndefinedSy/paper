---
type: concept
aliases: [Direct IO, O_DIRECT]
---

# Direct IO

## 定义

一种绕过操作系统 page cache 的 I/O 模式：应用程序直接与存储设备交换数据，不经过内核页缓存层的缓冲和回写管理。

## 核心要点

1. 优势：应用程序可以自行实现针对特定工作负载优化的缓存/写回策略，避免 page cache 的"双缓存"开销（数据同时存在于应用缓存和 page cache 中）与不可控的回写时机。
2. 代价：需要应用自己处理缓存管理、预读、写回调度等本来由内核负责的工作，实现复杂度上升。
3. 对延迟敏感、读写模式高度特化的存储系统（如数据库、KV 存储），自建缓存层通常优于依赖通用的 OS page cache。
4. [[SOS]] 支持 Direct IO 并自建类 pagecache 的写回管理机制（write cache + 异步 flush），用于精细控制 client 可见延迟与设备持久化的解耦。

## 代表工作

- [[SOS]]：磁盘存储层（dstorage）支持 Direct IO，配合专用后端线程的异步 flush 机制，使 client 可见写延迟不等待设备持久化。

## 相关概念

- [[Hash Index]]
- [[SSD]]
