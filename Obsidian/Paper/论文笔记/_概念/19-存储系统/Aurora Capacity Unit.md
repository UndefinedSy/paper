---
type: concept
aliases: [ACU]
---
# Aurora Capacity Unit
## 定义
Aurora Serverless 的容量计量单位：1 ACU ≈ 2 GB 内存及与之匹配的 CPU 与网络资源。
## 核心要点
1. 客户为整个 shard group 设定总的最小/最大 ACU，系统据**最大容量**推导初始 router 与 shard 数量（容量越高节点越多）。
2. 例：总 max ACU = 1200 → 初始 4 routers + 8 shards；2301–6144 区间 → 8 routers + 16 shards。
3. 每个节点另有自己的 min/max ACU，由 [[Proportional ACU Allocation]] 动态调整。
4. shard standby 的容量恒等于对应 primary，且**不计入**客户的总容量预算。
5. 论文实验中单节点最高可伸缩到 **256 ACU**（≈512 GB 内存）。
6. ACU 消耗曲线也是判断是否需要水平伸缩的核心监控信号（热点摊平的证据即来自它）。
## 代表工作
- [[AuroraLimitless]]：以 ACU 为伸缩与计费的统一抽象。
- [[Aurora Serverless V2]]：ACU 的定义来源。
## 相关概念
- [[Aurora Serverless V2]]
- [[Proportional ACU Allocation]]
- [[Amazon Aurora]]
