---
type: concept
aliases: [3FS, Fire-Flyer File System]
---

# DeepSeek 3FS

## 定义

DeepSeek 开源的面向 AI 训练场景的分布式文件系统，目标是提供极高的大规模顺序读写带宽（论文报告 6.6 TiB/s 级别），采用 [[CRAQ]] 作为复制协议以提升读吞吐。

## 核心要点

1. 设计目标与 [[SOS]] 不同：3FS 面向训练场景的海量顺序带宽需求（大批量数据批量读写），SOS 面向 AI serving 场景的单请求低尾延迟需求（P99.9 < 5ms）。
2. 采用 CRAQ 复制协议，允许从多副本读取以扩展读吞吐，这与 SOS 的 head-only 单跳读路径设计方向相反——SOS 为获得可预测的低单请求延迟牺牲了多副本读扇出能力。
3. 作为 KV cache 场景的候选存储后端出现在其他系统的基线对比中（如 PolarKV 论文中用 3FS 作为 storage-centric 基线）。

## 代表工作

- DeepSeek-AI 开源项目（2025）。
- [[SOS]]：相关工作中对比指出 3FS 针对训练场景的海量顺序带宽做优化，而 SOS 针对 AI serving 场景的单请求尾延迟做优化。

## 相关概念

- [[CRAQ]]
- [[Chain Replication]]
