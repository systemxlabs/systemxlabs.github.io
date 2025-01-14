+++
title = "DataFusion 查询引擎 Hash Join"
date = 2025-01-13
draft = true
+++

Hash Join

## 场景
主要用于 On 子句中没有等值条件的 Join 运算。例如默认用户配置下：表 `t0(a int, b int)` 和 `t1(c int, d int)`
1. `select * from t0 join t1 on t0.a = t1.c` 有一个 On 条件且是等值条件，走 Hash Join 算子
2. `select * from t0 join t1 on t0.a > t1.c` 有一个 On 条件但非等值条件，走 Nested Loop Join 算子
3. `select * from t0 join t1 on t0.a > t1.c and t0.b = t1.d` 有多个 On 条件且其中包含等值条件，走 Hash Join 算子

## 优化

## 执行
Hash Join 有两者执行模式：CollectLeft 和 Partitioned。Partitioned 模式要求左右表的 partition 数量相同。

### 第一阶段：build 阶段

CollectLeft 模式会将左表所有 partition 读取出来构建哈希表，而 Partitioned 模式会对左表每个 partition 单独构建哈希表。

### 第二阶段：probe 阶段

probe 阶段是分 partition 并行执行的，每个线程不断读取对应 partition 的右表数据，与左表数据进行 join。

## 哈希表


如何解决哈希冲突