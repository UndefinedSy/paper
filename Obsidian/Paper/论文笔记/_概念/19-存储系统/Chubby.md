---
type: concept
aliases: [Chubby Lock Service]
---

# Chubby

## 定义

Google 的分布式锁服务，为松耦合分布式系统提供锁定和一致性协调功能，存储 Bigtable 的启动元数据和复制配置。

## 核心要点

1. 提供分布式锁和少量元数据存储
2. [[Bigtable]] 使用 Chubby 存储 bootstrap 元数据（第一个 metadata tablet 的位置）
3. 复制配置存储在 Chubby 中，由 Bigtable master 协同更新
4. Tablet server 监视复制配置变更，异步传播（秒级）
5. 设计哲学：为一致性较高的场景提供可靠的协调服务

## 代表工作

- [[Bigtable]]: 存储启动元数据和复制配置

## 相关概念

- [[Bigtable]]
- [[Colossus]]
