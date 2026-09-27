---
type: concept
aliases: [可用区, AZ]
---
# Availability Zone

## 定义
Availability Zone（AZ，可用区）是云厂商在**同一地理区域内**划分的物理隔离单元：不同 AZ 拥有独立的供电、制冷与网络，因此可抵御单点机房故障，但彼此间延迟很低（通常在毫秒级）。

## 核心要点
1. **区域与 AZ 的分层**：一个 geo-region 包含多个 AZ。[[K2Plus]] 的默认部署是 3 个区域、每区域 8 台机器；层内存储池（[[DFV]]）的存储节点横跨 3 个 AZ。
2. **AZ 是 [[DFV]] 的冗余单位**：单 AZ 内靠多副本 + [[Erasure Coding|纠删码]]，跨 AZ 靠**同步写三副本**。DFV 三 AZ 全部落盘才 ACK，这是 11 个 9 持久性的来源。
3. **AZ 故障的影响**：[[K2Plus]] 定义了两类与 AZ 相关的区域故障——两 AZ 的 DFV 全部故障会导致区域存储池变只读；两 AZ 的 manager 集群故障会丧失管理能力。二者都需规划 [[Region Failover]]。
4. **管理面跨 AZ 部署**：[[Regional Manager]] 与 replicator manager 跨 AZ quorum 复制，AZ 故障时另一 AZ 副本晋升（实测 15 s 检测 + 提升，60 s 内数据节点全部恢复迁移）。
5. **与区域故障的区别**：AZ 故障由同区域其他 AZ 接管，恢复较快；区域故障需跨区域接管，若非活跃分区则面临 [[Cold Start]]。

## 代表工作
- [[K2Plus]]：AZ 故障实验中人为制造 AZ 间网络分区，验证 leader promotion、数据节点迁移与重新加入共识组的完整闭环。

## 相关概念
- [[DFV]]
- [[Regional Manager]]
- [[Region Failover]]
- [[Leader Election]]
- [[Erasure Coding]]
