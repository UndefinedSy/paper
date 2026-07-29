---
type: concept
aliases: [日志即数据, Log-as-Data]
---
# Log as Data
## 定义
将追加日志直接作为主数据存储，索引结构只保存指向日志数据块的引用，从而避免 WAL 与数据文件重复持久化完整 value。
## 核心要点
1. 消除同一 value 的双写，降低写放大和带宽占用。
2. 读取先查索引指针，再访问日志中的实际数据。
3. 日志本身仍需复制和校验，以维持 durability 与 crash consistency。
## 代表工作
- [[TokaDB]]：修改 LSM-tree entry，使其引用底层复制日志中的 KV Cache data block。
## 相关概念
- [[WAL]]
- [[LSM-tree]]
- [[ValueLog]]
