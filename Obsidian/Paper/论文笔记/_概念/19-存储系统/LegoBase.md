---
type: concept
aliases: [LegoBase, 内存解耦数据库]
---
# LegoBase

## 定义
LegoBase 是探索**内存解耦（memory disaggregation）**的数据库系统研究（VLDB 2021），把缓冲池与元数据等易失状态从 CPU 侧迁出，改由远程内存池通过 RDMA 承载，以提升弹性并加速恢复。

## 核心要点
1. **解耦对象是「内存」而非「存储」**：存储解耦（如 [[Disaggregated Storage-Compute Architecture]]）把持久数据放到共享存储池；内存解耦则进一步把**易失状态**也外置，让计算节点成为接近无状态的资源，可快速替换。
2. **技术前提是低延迟网络**：通常依赖 RDMA 或 CXL 织构（fabric），因为远程内存访问要走网络。
3. **局限**：[[K2Plus]] 在 Related Work 中指出，这类工作（LegoBase、PilotDB、PolarCXLMem、GaussDB）**大多假设单区域织构**，并未处理多区域事务处理的端到端挑战——把内存解耦与地理分布式事务语义结合仍是开放问题，超出 K2+ 的论文范围。

## 代表工作
- [[K2Plus]]：在 Related Work 的「Memory disaggregation」小节中引用，用于界定自身边界。

## 相关概念
- [[Disaggregated Storage-Compute Architecture]]
- [[DFV]]
- [[K2Plus]]
