---
type: concept
aliases: [租户级加密, Per-Tenant Encryption]
---

# Tenant Encryption

## 定义

以单个租户为粒度、使用可轮换的独立密钥对该租户数据进行加密的能力。在 SalesforceDB 中，通过将每个 [[LSM-tree|LSM]] fragment 限定为仅属于单一租户的数据，在 flush/compaction 时按租户密钥加密整个 fragment 实现。

## 核心要点

1. 前提条件：租户聚簇存储（tenant-clustered storage）保证单个 fragment 内的数据都属于同一租户
2. 加密/解密发生在 fragment 边界，密钥 ID 嵌入 fragment 元数据中；访问（查询、compaction 等）时按嵌入的 key ID 自动解密
3. 密钥轮换由 [[KMS]]（密钥管理系统）触发，通过 LSM compaction 完成数据重加密；轮换完成前旧密钥加密的数据依然可读
4. 得益于不可变持久化：传统可变页引擎每次页修改和驱逐都要重新加密整页，而 SalesforceDB 的加解密只需在 fragment 首次写入和后续读取时发生，开销极低（实测加密 13.96μs/fragment、解密 12.52μs/fragment，相对 ms 级总读写时间可忽略）
5. 与 Shield 等同样在 LSM flush/compaction 边界加密的方案不同，SalesforceDB 支持真正的按租户独立密钥，而非统一密钥

## 代表工作

- [[SalesforceDB]]：生产环境加密超过 7,000 个租户（最大 80+TB），是本文"不可变性带来跨层收益"的典型案例之一

## 相关概念

- [[LSM Virtualization]]
- [[Immutability]]
- [[KMS]]
- [[LSM-tree]]
