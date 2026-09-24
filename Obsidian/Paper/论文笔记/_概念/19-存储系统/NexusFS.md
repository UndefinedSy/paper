---
type: concept
aliases: [计算侧虚拟文件系统]
---

# NexusFS

## 定义

ByteHouse 的 alignment-aware 虚拟文件系统，把远端 CrossCache、计算节点本地 SSD 与内存缓冲统一为对齐的数据访问路径。

## 核心要点

1. region、disk segment 与 memory buffer 端到端对齐。
2. region manager 负责映射和 FIFO 回收，buffer manager 使用 second-chance。
3. 保留 segment 可零拷贝暴露给执行流水线。

## 代表工作

- [[ByteHouse]]：在实时多模态云原生数仓中实现或评估该概念。

## 相关概念

- [[CrossCache]]
- [[Shared-Storage Architecture]]
