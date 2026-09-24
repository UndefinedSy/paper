---
type: concept
aliases: [LevelDB, Level DB, Google LevelDB]
---

# LevelDB

## 定义

Google 开源的嵌入式键值存储库，基于 [[LSM-tree]] 实现，以顺序写入换取高吞吐量，主要用于本地持久化场景，是 Firestore iOS 客户端 SDK 本地文档缓存的底层实现。

## 核心要点

1. **LSM-tree 架构**：写入先到内存中的 MemTable，满后刷到磁盘 SSTable，后台 Compaction 合并层级；顺序写性能远优于随机写，适合日志型与缓存型工作负载。
2. **嵌入式设计**：作为库直接链接进应用进程，无独立服务进程，启动开销极低，特别适合移动端 SDK 和桌面客户端。
3. **Firestore SDK 中的作用**：在 iOS 上存储两区文档缓存（已确认区 + 待提交变更区）、本地查询索引，以及缓存管理的辅助数据；支持 SDK 的离线查询推测执行与重连后的自动对账。
4. **与同类方案对比**：Android Firestore SDK 用 SQLite（关系型，支持复杂查询），浏览器端用 IndexedDB（浏览器原生 KV 存储）；LevelDB 在 iOS 上负担 KV 访问即可，不需要关系查询。

## 代表工作

- [[Firestore]]: The Live Database: Firestore's Scalable and Consistent Realtime Queries (PVLDB 2026)
- Google. LevelDB. 2014. https://github.com/google/leveldb

## 相关概念

- [[LSM-tree]]
- [[SSTable]]
- [[WAL]]
