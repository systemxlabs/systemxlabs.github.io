+++
title = "Ballista 分布式查询引擎 - 分布式执行计划"
date = 2024-04-19
draft = false
+++

**原始 SQL**
```sql
SELECT customer.c_custkey, sum(orders.o_totalprice) as total_amount 
FROM customer JOIN orders ON customer.c_custkey = orders.o_custkey 
GROUP BY customer.c_custkey;
```
注：customer 和 orders 两张表数据均分为 3 个 parquet 文件存储。

**使用 Datafusion 生成单机执行计划**
```
ProjectionExec: expr=[c_custkey@0 as c_custkey, SUM(orders.o_totalprice)@1 as total_amount]
  AggregateExec: mode=SinglePartitioned, gby=[c_custkey@0 as c_custkey], aggr=[SUM(orders.o_totalprice)]
    ProjectionExec: expr=[c_custkey@0 as c_custkey, o_totalprice@2 as o_totalprice]
      HashJoinExec: mode=Partitioned, join_type=Inner, on=[(c_custkey@0, o_custkey@0)]
        RepartitionExec: partitioning=Hash([c_custkey@0], 16), input_partitions=16
          RepartitionExec: partitioning=RoundRobinBatch(16), input_partitions=3
            ParquetExec: file_groups={3 groups: [...]}, projection=[c_custkey]
        RepartitionExec: partitioning=Hash([o_custkey@0], 16), input_partitions=16
          RepartitionExec: partitioning=RoundRobinBatch(16), input_partitions=3
            ParquetExec: file_groups={3 groups: [...]}, projection=[o_custkey, o_totalprice]
```

为什么会生成这样的单机执行计划？
1. Datafusion 提供了 target_partitions 配置项（默认为本机的 CPU 核心数量）来配置并行度，在生成单机执行计划时会插入 RepartitionExec 算子来调整 partition 数量。由于表数据分布在 3 个 parquet 文件，
ParquetExec 读取后输出 3 个 partition，因此在这里插入 RepartitionExec 算子将 partition 数量从 3 个提高到 16 个。（在分布式环境下为了利用多个机器，支持更高的并行度，Ballista 提供了可以给每个 session 手动配置此项的支持）
2. Datafusion 提供了 repartition_joins 开关项，基于 join key 进行 hash repartition 后可并行执行 hash join。


**生成初步的分布式执行计划**
```
=========ResolvedStage[stage_id=1.0, partitions=3]=========
ShuffleWriterExec: Some(Hash([Column { name: "c_custkey", index: 0 }], 16))
  ParquetExec: file_groups={3 groups: [...]}, projection=[c_custkey]

=========ResolvedStage[stage_id=2.0, partitions=3]=========
ShuffleWriterExec: Some(Hash([Column { name: "o_custkey", index: 0 }], 16))
  ParquetExec: file_groups={3 groups: [...]}, projection=[o_custkey, o_totalprice]

=========UnResolvedStage[stage_id=3.0, children=2]=========
ShuffleWriterExec: None
  ProjectionExec: expr=[c_custkey@0 as c_custkey, SUM(orders.o_totalprice)@1 as total_amount]
    AggregateExec: mode=SinglePartitioned, gby=[c_custkey@0 as c_custkey], aggr=[SUM(orders.o_totalprice)]
      ProjectionExec: expr=[c_custkey@0 as c_custkey, o_totalprice@2 as o_totalprice]
        HashJoinExec: mode=Partitioned, join_type=Inner, on=[(c_custkey@0, o_custkey@0)]
          UnresolvedShuffleExec
          UnresolvedShuffleExec
```

在 stage1 和 stage2 执行完毕后，stage3 会更新成如下
```
ShuffleWriterExec: None
  ProjectionExec: expr=[c_custkey@0 as c_custkey, SUM(orders.o_totalprice)@1 as total_amount]
    AggregateExec: mode=SinglePartitioned, gby=[c_custkey@0 as c_custkey], aggr=[SUM(orders.o_totalprice)]
      ProjectionExec: expr=[c_custkey@0 as c_custkey, o_totalprice@2 as o_totalprice]
        HashJoinExec: mode=Partitioned, join_type=Inner, on=[(c_custkey@0, o_custkey@0)]
          ShuffleReaderExec: partitions=16
          ShuffleReaderExec: partitions=16
```
UnsolvedShuffleExec 会被 ShuffleReaderExec 算子替代。

**最终的分布式执行计划**
![ballista-mvp-dag](./ballista-mvp-dag.drawio.png)

为什么会生成这样的分布式执行计划？
1. Ballista 会在执行 repartition 的算子（如 RepartitionExec/CoalescePartitionsExec/SortPreservingMergeExec 算子，也被称为 pipeline breaker）那里插入 shuffle 算子，将单机执行计划分割成多个 stage，每个 stage 内部所有算子均为相同的分区方案。
2. 每个 stage 最终都会通过 ShuffleWriterExec 算子将执行结果 repartition 并写入本地磁盘。
3. 每个有前置依赖的 stage 都会从 ShuffleReaderExec 算子开始执行，ShuffleReaderExec 算子负责读取前置 stage 产生的中间执行结果。
