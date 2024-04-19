+++
title = "Ballista 分布式查询引擎 - 分布式执行计划"
date = 2024-04-19
draft = true
+++

原始 SQL
```sql
SELECT customer.c_custkey, sum(orders.o_totalprice) as total_amount 
FROM customer JOIN orders ON customer.c_custkey = orders.o_custkey 
GROUP BY customer.c_custkey;
```
注：customer 和 orders 两张表数据均分为 3 个 parquet 文件存储。

使用 Datafusion 生成单机执行计划
```
ProjectionExec: expr=[c_custkey@0 as c_custkey, SUM(orders.o_totalprice)@1 as total_amount]
  AggregateExec: mode=SinglePartitioned, gby=[c_custkey@0 as c_custkey], aggr=[SUM(orders.o_totalprice)]
    ProjectionExec: expr=[c_custkey@0 as c_custkey, o_totalprice@2 as o_totalprice]
      CoalesceBatchesExec: target_batch_size=8192
        HashJoinExec: mode=Partitioned, join_type=Inner, on=[(c_custkey@0, o_custkey@0)]
          CoalesceBatchesExec: target_batch_size=8192
            RepartitionExec: partitioning=Hash([c_custkey@0], 16), input_partitions=16
              RepartitionExec: partitioning=RoundRobinBatch(16), input_partitions=3
                ParquetExec: file_groups={3 groups: [...]}, projection=[c_custkey]
          CoalesceBatchesExec: target_batch_size=8192
            RepartitionExec: partitioning=Hash([o_custkey@0], 16), input_partitions=16
              RepartitionExec: partitioning=RoundRobinBatch(16), input_partitions=3
                ParquetExec: file_groups={3 groups: [...]}, projection=[o_custkey, o_totalprice]
```
为什么会生成这样的单机执行计划？
1. xxx


生成初步的分布式执行计划
```
=========ResolvedStage[stage_id=1.0, partitions=3]=========
ShuffleWriterExec: Some(Hash([Column { name: "c_custkey", index: 0 }], 16))
  ParquetExec: file_groups={3 groups: [...]}, projection=[c_custkey]

=========ResolvedStage[stage_id=2.0, partitions=3]=========
ShuffleWriterExec: Some(Hash([Column { name: "o_custkey", index: 0 }], 16))
  ParquetExec: file_groups={3 groups: [...]}, projection=[o_custkey, o_totalprice]

=========UnResolvedStage[stage_id=3.0, children=2]=========
Inputs{1: StageOutput { partition_locations: {}, complete: false }, 2: StageOutput { partition_locations: {}, complete: false }}
ShuffleWriterExec: None
  ProjectionExec: expr=[c_custkey@0 as c_custkey, SUM(orders.o_totalprice)@1 as total_amount]
    AggregateExec: mode=SinglePartitioned, gby=[c_custkey@0 as c_custkey], aggr=[SUM(orders.o_totalprice)]
      ProjectionExec: expr=[c_custkey@0 as c_custkey, o_totalprice@2 as o_totalprice]
        CoalesceBatchesExec: target_batch_size=8192
          HashJoinExec: mode=Partitioned, join_type=Inner, on=[(c_custkey@0, o_custkey@0)]
            CoalesceBatchesExec: target_batch_size=8192
              UnresolvedShuffleExec
            CoalesceBatchesExec: target_batch_size=8192
              UnresolvedShuffleExec
```
