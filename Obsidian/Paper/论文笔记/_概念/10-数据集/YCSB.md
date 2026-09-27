---
type: concept
aliases: [YCSB, Yahoo! Cloud Serving Benchmark]
---
# YCSB

## 定义
YCSB（Yahoo! Cloud Serving Benchmark）是评测键值/云存储系统的标准基准，核心是一组简单读写操作（read/update/insert/scan）与可配置的工作负载参数。

## 核心要点
1. **可调维度**：(1) 读写比例（如 10%/50%/90% 写）；(2) 数据访问倾斜度，用 **Zipf 系数** $\theta$ 控制——$\theta$ 越大越倾斜，热点越集中；(3) 请求分布（uniform / zipfian / latest）。
2. **价值在于敏感性分析**：标准事务基准（如 [[TPC-C]]）模拟固定业务，YCSB 则让你扫描**参数空间**，观察系统在冲突率与倾斜度变化下的退化曲线。
3. **[[K2Plus]] 中的用法**：用 YCSB 变化 Zipf 系数（0.2/0.5/0.8）与写比例（10%/50%/90%），并在 failover 研究中使用**固定开环（open-loop）负载**以量化恢复时间（定义为吞吐恢复到崩溃前稳态的 95%）。
4. **相关变体**：YCSB+T 增加了事务支持（论文撰写者之一 Röhm 的工作）。

## 代表工作
- [[K2Plus]]：YCSB 用于敏感性分析（Figure 16）与全部故障注入实验的负载生成。

## 相关概念
- [[TPC-C]]
- [[K2Plus]]
