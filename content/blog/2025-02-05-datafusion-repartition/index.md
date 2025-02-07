+++
title = "DataFusion 查询引擎 Repartition"
date = 2025-02-05
draft = true
+++

DataFusion 采用的执行模型是基于异步 Stream 的火山模型（Pull Based），Repartition（Exchange Operator）是其并行执行的关键。Repartition 的作用就是将输入的 N 个 partitions 按照指定分区方案重新分区，输出 M 个 partitions。