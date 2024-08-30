+++
title = "DataFusion 两阶段并行哈希分组聚合"
date = 2024-08-23
draft = true
+++

分组聚合功能是任何分析引擎的核心功能，可在海量数据上创建出可以理解的摘要。DataFusion 分析引擎采用了先进的两阶段并行哈希分组聚合技术，高度并行且向量化执行。

## 多种聚合方案
DataFusion 支持多种聚合方案，在不同情况下会选择最优方案。

### 一阶段无哈希分组（Single）
![](./datafusion-aggregation-single.drawio.png)
- Aggregate 算子接收所有输入数据串行执行
- 一个 Aggregate 算子完成所有分组聚合工作
- 场景：通常输入只有一个 partition

### 一阶段有哈希分组（SinglePartitioned）
![](./datafusion-aggregation-single-partitioned.drawio.png)
- 输入必须按照 group key 进行了重新分区 
- 一个 Aggregate 算子完成所有分组聚合工作
- Aggregate 算子接收多个 partition 数据并行执行
- 场景：通常输入有多个 partitions，且都已经按 group key 重新分区（repartition）了

### 两阶段无哈希分组（Partial-Final）
![](./datafusion-aggregation-partial-final.drawio.png)
- 第一阶段 Aggregate 算子接收多个 partition 数据并行执行，计算中间聚合结果
- 第二阶段 Aggregate 算子接收所有中间聚合结果数据串行执行，生成最终聚合结果
- 场景：通常输入有多个 partitions，查询没有 group by 语句 或者 用户设置并行度为 1（输出一个 partition）

### 两阶段有哈希分组（Partial-FinalPartioned）
![](./datafusion-aggregation-partial-final-partitioned.drawio.png)
- 第一阶段 Aggregate 算子接收多个 partition 数据并行执行，计算中间聚合结果
- 第二阶段 Aggregate 算子接收多个 partition 中间聚合结果数据并行执行，并行生成最终聚合结果
- 第二阶段输入必须按照 group key 进行了重新分区（repartition）
- 场景：通常输入有多个 partitions（没有按 group key 重新分区），查询有 group by 语句，且并行度大于 1

## 两阶段并行哈希分组聚合（Partial-FinalPartioned）

第一阶段算子输出 schema 跟第二阶段可能有所不同，例如 `select a, avg(b) from t group by a`，在第二阶段输出的是 `(a, avg(b))`，但在第一阶段输出的是 `(a, count(b), sum(b))`（聚合计算的中间状态）。

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