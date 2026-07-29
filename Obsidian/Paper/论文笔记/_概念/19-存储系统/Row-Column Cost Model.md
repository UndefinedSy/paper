---
type: concept
aliases: [行列格式成本模型, Storage Format Cost Model]
---

# Row-Column Cost Model

## 定义

ByteGraph-Dione 用于在线比较单条邻接表在行存与列存下执行当前查询窗口总成本的简化模型。

## 数学形式

$$
Cost_f=\sum_i\left(a_fN_{rows,i}N_{attr,i}+BN_{rows,i}+C_f\right)
$$

其中列格式会进一步按 OLTP 与 OLAP 使用不同的单位系数和固定成本。

## 核心要点

1. $N_{rows}$ 和 $N_{attr}$ 捕获扫描范围与投影宽度。
2. $B$ 表示页内定位与版本搜索成本。
3. $C$ 汇总页位置、cache 与 I/O 等固定开销。
4. 参数来自生产 trace 的采样查询平均执行时间。
5. 只有收益超过转换阈值才触发在线格式切换。

## 代表工作

- [[ByteGraph-Dione]]：用该模型驱动邻接表级自适应行列转换。

## 相关概念

- [[Adaptive Dual-Format Graph Storage]]
- [[MVCC]]
- [[Adjacency List]]
