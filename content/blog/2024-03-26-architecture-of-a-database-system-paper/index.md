+++
title = "数据库系统架构论文"
date = 2024-03-26
+++

[论文](./architecture-of-a-database-system.pdf) | [中文翻译](./architecture-of-a-database-system-chinese.pdf)

## DBMS 主要组件
![main-components-of-dbms](main-components-of-dbms.png)

Process Manager 并非单指分配进程，而是根据 DBMS 实际实现的进程模型，分配进程或线程。其作用之一 Admission Control 指是否立即处理该查询，或是等待系统有足够资源时再处理。

## 进程模型
### 每个 DBMS Worker 一个进程
- 优点：可移植性好（早期的 OS 对线程支持较差），可以利用 OS 保护措施
- 缺点：进程切换代价更大
- 案例：DB2，PostgreSQL，Oracle

### 每个 DBMS Worker 一个线程
线程又可分为 OS线程 或 DBMS 线程（轻量级线程，仅在用户空间调度而没有内核调度程序的参与）。

- 优点：线程切换代价较小，共享数据友好
- 缺点：OS 对线程不提供溢出和指针的保护，调试困难，可移植性较差（在当时）
- 案例：DB2，SQL Server，MySQL

### 进程/线程池

由中央进程/线程控制所有客户端连接，由进程/线程池管理所有 DBMS worker。大小可动态变化。

DBMS worker 之间会共享数据
1. 磁盘缓冲池（buffer pool）
2. 锁表

DBMS 可能支持多种进程模型，比如 DB2 支持以上四种，Oracle 在 Windows 系统上采用多线程模型。

**准入控制**
1. 通过调度进程/线程来确保客户端连接数在一个临界值以下
2. 在查询语句转换和优化后的执行阶段，通过分析查询所需资源来决定是否推迟查询执行

## 并行架构

### 共享内存（Shared-Memory）
<img src="shared-memory-architecture.png" alt="shared-memory-architecture" width="400"/>

在共享内存机器，OS 通常支持作业（进程或线程）被透明地分配到每个处理器上，并且共享的数据结构可以继续被所有作业所访问，所以上面四种进程模型可以运行良好。主要需要修改查询执行层，将单一的 SQL 并行到多个处理器上。

### 无共享（Shared-Nothing）
<img src="shared-nothing-architecture.png" alt="shared-nothing-architecture" width="400"/>

通过网络互联通信。表数据被水平分区到不同机器，对于每个 SQL 请求，会被发送到集群其他成员，然后各自并行执行查询本地所保存的数据。事务实现较复杂。可扩展性强。需要冗余数据来提高可用性。

### 共享磁盘（Shared-Disk）
<img src="shared-disk-architecture.png" alt="shared-disk-architecture" width="400"/>

单个 DBMS 执行节点发生故障不会影响其他节点访问整个数据库。需要手动协调多个节点的共享数据（分布式锁，分布式缓冲池）。

### NUMA（Non-Uniform Memory Access）
<img src="numa-architecture.png" alt="numa-architecture" width="400"/>

处理器访问本地内存比远程内存更快。NUMA 允许共享内存系统扩展到更多数量处理器的规模。

## 关系查询处理器