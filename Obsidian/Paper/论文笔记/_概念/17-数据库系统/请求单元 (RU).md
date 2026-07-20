---
type: concept
aliases: [Request Unit, RU, 请求单元]
---

# 请求单元 (RU)

## 定义

Request Unit (RU) 是 Serverless 数据库中用于抽象化资源消耗的计费单位，将 CPU、内存、磁盘 I/O 等底层资源统一量化，使用户只需关注请求吞吐量而非基础设施细节。

## 数学形式

**写操作**：
$$RU_{write} = S_{write} / U, \quad \text{总费用} = r \cdot RU_{write}$$

**读操作（ABase 缓存感知版本）**：
$$RU_{read} = \mathbb{E}[S_{read}] \times (1 - \mathbb{E}[R_{hit}]) / U$$

其中 $U = 2\text{KB}$（经验单位）

## 核心要点

1. 抽象底层硬件，让用户以"请求量"为单位管理资源，而非 CPU/内存/磁盘
2. 可用于计费和隔离两种目的
3. ABase 的创新：将缓存命中率 $\mathbb{E}[R_{hit}]$ 纳入读操作 RU 计算，命中代理缓存的请求不计费

## 代表工作

- [[ABase]]: 缓存感知 RU，结合代理层和 DataNode 层缓存命中率修正
- [[DynamoDB]]: 最早将 RU 推广为 Serverless KV 数据库核心抽象

## 相关概念

- [[WFQ|Weighted Fair Queueing]]
- [[Resource Pool|资源池]]
