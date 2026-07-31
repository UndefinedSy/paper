---
type: concept
aliases: [谓词下推]
---
# Predicate Pushdown
## 定义
查询优化技术：把过滤条件尽可能推到靠近数据的一侧求值，以减少需要传输与处理的数据量。
## 核心要点
1. 在分布式数据库中，它决定了 router 与 shard 之间的网络流量规模。
2. [[Aurora Limitless Database]] 把谓词分为两类：**foreign 谓词**（可安全在 shard 求值，如只含内建操作与 `IMMUTABLE` 函数）与 **local 谓词**（不安全，需拉数据到 router）。
3. local 谓词的两个来源：含可变函数（`STABLE`、`VOLATILE`）的谓词；需以不同用户身份执行的 definer 函数。
4. 这个「安全性判定」是正确性要求而非纯优化——在 shard 上求值可变函数可能得到不同结果。
## 代表工作
- [[AuroraLimitless]]：foreign/local 谓词二分与下推。
## 相关概念
- [[Foreign Data Wrapper]]
- [[Join Pushdown]]
- [[Partition Pruning]]
- [[PostgreSQL]]
