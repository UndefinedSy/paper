---
type: concept
aliases: [Neon VM]
---

# NeonVM

## 定义

[[Neon]] 提出的 API：**在 Kubernetes 中运行虚拟机**，支持**原地垂直扩容**（in-place vertical scaling）与**热迁移**（live migration），无需重启 VM。每个计算作为一个独立的 QEMU/KVM VM 运行。

## 核心要点

1. **为什么不用原生容器**：① 宿主–容器隔离弱，是安全漏洞的主要来源；② 云托管 K8s 中供给新资源（某些 VM）要几十到几百秒，不适合 [[Scale-to-Zero]] 场景。
2. **CPU 伸缩**：通过 guest agent 在 guest OS 内**启停单个 CPU**。
3. **内存伸缩**：使用 **virtio-mem**，平衡 movable 与 kernel 内存的比例；并**限制伸缩范围**以避免碎片导致的降配失败。
4. **磁盘伸缩**：结合**协作式回收**（文件系统发 discard 操作 → 在宿主后备文件上转化为 hole punch）与对 swap / 临时存储的**超分**。
5. **缓存同步调整**：盘支撑的计算缓存（[[Local File Cache]]）在伸缩操作中被**同步**调整大小。
6. **配套的池化**：[[Warm Pool]] 中的 VM 规格统一，靠原地垂直扩容在分配时适配用户配置。
7. **伸缩单元**：以 1 GiB RAM 及其成比例的 CPU 份额为一个 scaling unit，线性且均匀；CPU / 内存 / 缓存目标独立估算后取最大值作为 VM 尺寸。

## 代表工作

- [[Neon]]: NeonVM 的提出者。
- [[Lakebase]]: 计算虚拟化基础，承载每天数百万次伸缩操作；论文指出这些"战略选择"是为了限制维护负担（例如让缓存走 OS page cache 而非普通分配）。

## 相关概念

- [[Scale-to-Zero]]
- [[Warm Pool]]
- [[Local File Cache]]
- [[Elastic Pool]]
