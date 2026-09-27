---
type: concept
aliases: [分区拆分]
---
# Partition Split

## 定义
Partition Split 是把一个过载或过大的数据分区**按 key 范围一分为二**的操作，两个子分区各持独立的存储对象（Root/Index/WAL Plog 与 LSM 树）。

## 核心要点
1. **触发条件**：数据节点接近过载（如 CPU ≥90%），或热点分区导致单节点资源耗尽。
2. **[[K2Plus]] 的拆分流程**（Figure 12）：
   - ① [[Regional Manager]] 选择拆分点并更新路由；
   - ② 旧分区暂停、把已清理的 segment flush 到 [[DFV]] 并阻塞新 [[WAL]] 写；
   - ③ 创建新的存储对象；
   - ④ 既有 SSTable 由新分区的 index Plog 重新索引。
3. **迁移复用**：拆分后的迁移沿用分区迁移流程，但**目的节点跳过 WAL 填充/flush**（数据已在 ② 阶段落盘），只复制索引与元数据——这是拆分+迁移能在数秒内完成的原因。
4. **区域自治红利**：由于分区元数据不跨区域，其他区域**无需镜像**这次拆分，不涉及任何全局协调或跨区域通信。
5. **细粒度均衡**：每个 CPU 核可承载多个物理分区，因此可以做**分区级**而非整 shard 级的迁移，把负载摊平到各核。

## 代表工作
- [[K2Plus]]：实测拆分与迁移 3 s 内完成，聚合吞吐提升 5.7×。

## 相关概念
- [[Partition Migration]]
- [[Physical Partition]]
- [[Regional Manager]]
- [[Load Balancing]]
