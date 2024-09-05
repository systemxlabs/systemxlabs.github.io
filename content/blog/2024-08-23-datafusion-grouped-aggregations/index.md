+++
title = "DataFusion 两阶段并行哈希分组聚合"
date = 2024-08-23
draft = true
+++

分组聚合功能是任何分析引擎的核心功能，可在海量数据上创建出可以理解的摘要。DataFusion 分析引擎采用了先进的两阶段并行哈希分组聚合技术，高度并行且向量化执行。

## 多种聚合方案
DataFusion 支持多种聚合方案，在不同情况下会选择最优方案。

### 一阶段无哈希分区（Single）
![](./datafusion-aggregation-single.drawio.png)
- Aggregate 算子接收所有输入数据串行执行
- 一个 Aggregate 算子完成所有分组聚合工作
- 场景：通常输入只有一个 partition

### 一阶段有哈希分区（SinglePartitioned）
![](./datafusion-aggregation-single-partitioned.drawio.png)
- 输入必须按照 group key 进行了重新分区 
- 一个 Aggregate 算子完成所有分组聚合工作
- Aggregate 算子接收多个 partition 数据并行执行
- 场景：通常输入有多个 partitions，且都已经按 group key 重新分区（repartition）了

### 两阶段无哈希分区（Partial-Final）
![](./datafusion-aggregation-partial-final.drawio.png)
- 第一阶段 Aggregate 算子接收多个 partition 数据并行执行，计算中间聚合结果
- 第二阶段 Aggregate 算子接收所有中间聚合结果数据串行执行，生成最终聚合结果
- 场景：通常输入有多个 partitions，查询没有 group by 语句 或者 用户设置并行度为 1（输出一个 partition）

### 两阶段有哈希分区（Partial-FinalPartioned）
![](./datafusion-aggregation-partial-final-partitioned.drawio.png)
- 第一阶段 Aggregate 算子接收多个 partition 数据并行执行，计算中间聚合结果
- 第二阶段 Aggregate 算子接收多个 partition 中间聚合结果数据并行执行，并行生成最终聚合结果
- 第二阶段输入必须按照 group key 进行了重新分区（repartition）
- 场景：通常输入有多个 partitions（没有按 group key 重新分区），查询有 group by 语句，且并行度大于 1

## 两阶段并行哈希分组聚合（Partial-FinalPartioned）

以 `select a, b, avg(c) from t group by a, b` 为例。

### 第一阶段（Partial）
- 跳过聚合
  - 为什么必须 group ordering 为 none


1. 从输入读取一批数据 `(a, b, c)`
2. 求值 group keys `a` 和 `b` 和 聚合函数入参 `c`
3. 基于分组和聚合函数执行聚合计算（并非计算最终 `avg` 结果，而是计算中间结果 `count(c)` 和 `sum(c)`）
4. 计算是否为高基数聚合（group 非常分散，默认阈值是行数大于 100000 并且 group 数量与行数比值大于 0.8）
5. 判断 group 数量是否达到 limit，如果达到则停止读取输入数据，改为输出中间聚合结果 `(a, b, count(c), sum(c))`，随即此阶段完成
6. 判断是否可以输出部分中间聚合结果（如果输入已按照 group keys 排好序，则可以预见后续输入不会再出现同一 group 值，此时可以将该 group 输出）
    1. 停止读取输入数据，改为输出部分中间聚合结果
    2. 输出完毕后，继续读取输入数据（跳到第 1 步）
7. 如果内存已经不足，则提前输出（early emit）中间聚合结果，输出完毕后，继续读取输入数据（跳到第 1 步）
8. 判断是否跳过聚合计算（基于第 4 步结果），如果跳过
    1. 停止读取输入数据，改为输出已经聚合的全部中间结果
    2. 读取输入数据，不做聚合计算直接将输入转换为中间结果 `(a, b, count(c), sum(c))` 输出，直至输入完毕，随即此阶段完成
9. 继续第 1 步，直至输入完毕，然后输出所有中间结果，随即此阶段完成


### 第二阶段（FinalPartitioned）
- 输出schema
- 溢出到磁盘
  - 排序

1. 从输入读取一批中间聚合数据 `(a, b, count(c), sum(c))`，判断分配的内存是否还有空间，如果没有则溢出到磁盘
   1. 将内存中所有中间聚合数据按照 group keys 升序排序 `[a asc, b asc]`
   2. 以 Arrow IPC 格式写入磁盘文件
2. a


第一阶段算子输出跟第二阶段可能有所不同，例如 `select a, avg(b) from t group by a`，在第二阶段输出的是最终结果 schema `(a, avg(b))`，但在第一阶段输出的是聚合计算的中间状态 `(a, count(b), sum(b))`。

三种情况
1. no group
2. group + limit => topk
3. group

问题：
1. 如何向量化执行的
2. 内部hash表结构
3. Final 和 FinalPartitioned
4. 何时使用两阶段、何时使用一阶段

参考
1. https://arrow.apache.org/blog/2023/08/05/datafusion_fast_grouping/