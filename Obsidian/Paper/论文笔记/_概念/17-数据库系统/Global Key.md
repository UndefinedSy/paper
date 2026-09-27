---
type: concept
aliases: [Global Key, 全局键]
---
# Global Key

## 定义
Global Key 是 **[[K2Plus]] 表主键的一个子集**（不要求唯一），作为全局数据放置的**粗粒度局部性提示**，让应用层语义引导全局 range 的分布。

## 核心要点
1. **桥接逻辑语义与物理放置**：例如纽约的子项目管理部门可用 location 属性（如 302）作 Global Key，把主数据副本放在对应区域以改善局部性。
2. **路由粒度**：Global Key 决定全局路由（选哪个区域），主键的**剩余部分**决定区域内的路由（[[Regional Manager|regional routing]]）。
3. **可省略**：若表中记录无强局部性语义，用户可不声明 Global Key 与 global range map，此时表被视为驻留在创建它的区域，系统自动管理下层区域内分区。
4. **DDL 形态**：`GLOBAL KEY (C1)` + `GLOBAL RANGE MAP (""::p[R1]::r[R2,R3], "100"::p[R2]::r[R1,R3], ...)`，其中 `p` 是主区域、`r` 是副本区域集合。
5. **不用于常规负载均衡**：K2+ 正常运行时主要靠区域内资源弹性做均衡，而非全局键重分区。

## 代表工作
- [[K2Plus]]：Global Key + Global Range Map 是两层路由的入口定义。

## 相关概念
- [[Global Manager]]
- [[Regional Manager]]
- [[Replication Middleware]]
