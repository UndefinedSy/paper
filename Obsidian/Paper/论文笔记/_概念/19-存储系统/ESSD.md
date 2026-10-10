---
type: concept
aliases: [ESSD, Enhanced SSD, Alibaba Cloud Block Storage]
---

# ESSD

## 定义

ESSD（Enhanced SSD）是阿里云提供的云块存储（CBS）产品，分为 PL0/PL1/PL2/PL3 多个性能档位，带宽随所购容量强耦合扩展。

## 核心要点

1. **带宽–容量耦合**：各档位的 provisioned bandwidth 由容量 $C$ 决定（如 PL0: $\min(100+0.25C, 180)$ MB/s），要获得高带宽往往需要购买远超实际需求的容量。
2. **每 GB/s 带宽成本随档位上升**：PL3 虽然带宽上限最高（4000 MB/s），但其每 GB/s 带宽成本（7946 CNY/月）反而是四个档位中最贵的，PL0 最便宜（910 CNY/月）。
3. CBS 必须挂载在网络带宽足够的 VM 上，高带宽存储部署往往要求高规格 VM，进一步推高总成本。

## 代表工作

- [[PolarKV]]：基于 ESSD 的带宽–容量–成本特性，选用多块低成本 PL0 ESSD 做 [[RAID-0]] 聚合带宽，而非直接采购高档位 ESSD。

## 相关概念

- [[RAID-0]]
- [[Cloud Block Storage]]（概念：云块存储的通用抽象，可按需补充）
