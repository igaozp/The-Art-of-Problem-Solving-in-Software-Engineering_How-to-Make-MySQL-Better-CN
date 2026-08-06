# 第2章：神秘的 MySQL 问题

MySQL 是最流行的开源数据库之一，历经数十年发展，以简单、易用著称，常被互联网公司选作基础组件。即便如此，它仍面临大量挑战。

本章用九个容易令人困惑的问题或现象作引子，为后续深入讨论做铺垫。

## 2.1 SysBench 读写测试表现出超线性吞吐量增长

以 MySQL 8.0.27 发布版为例：在 x86 四路 NUMA 环境下，用 SysBench 远程测试读写；隔离级别为读已提交（Read Committed）；同机部署实例 1 与实例 2，单次测试 60 秒。两实例分别单独压测的结果如下。

<img src="../media/image-20240829081346732.png" alt="image-20240829081346732" style="zoom:150%;" />

图 2-1. MySQL 单独运行时的吞吐量。

单独运行时吞吐都不高，分别为 172,781 QPS 与 155,387 QPS，合计 328,168 QPS。若用 SysBench 同时压这两个实例，则分别达到 271,232 QPS 与 275,197 QPS。

<img src="../media/image-20240829204745429.png" alt="image-20240829204745429" style="zoom:150%;" />

图 2-2. MySQL 一起运行时的吞吐量。

同时压测时两实例合计 546,429 QPS，明显高于“各自单独跑再相加”的结果。统计对比见下图。

<img src="../media/image-20240829081541149.png" alt="image-20240829081541149" style="zoom:150%;" />

图 2-3. 单独运行与一起运行的总吞吐量。

从数量关系看：两实例同机合计约等于各自单独吞吐之和，可视为近似线性；若合计明显高于该和，则表现为超线性。

是什么导致超线性？MySQL 是否真的“超线性”？要回答这些问题，需要结合计算机基础与更深的 MySQL 机制。

## 2.2 应用 PGO 后 TPC-C 吞吐量意外下降

配置文件引导优化（Profile-Guided Optimization，PGO）通过插桩或采样收集运行特征，再据此改进编译期优化决策 [45]。它理论上很有效，但双重编译流程繁琐，因而在工程中普及有限。即便如此，PGO 仍值得用于提升 MySQL 性能。

下图概括在较高版本 MySQL 8.0 上应用 PGO 的流程。

![](../media/cb771856f5b0a021a92acc162c9c13bd.png)

图 2-4. 在更高版本的 MySQL 8.0 中使用 PGO：分步指南。

如图所示，PGO 大致分三步：

1. 使用 *"-DFPROFILE_GENERATE=ON"* 编译目标版本的 MySQL。
2. 启动该版本，用 TPC-C 等负载采集训练数据与性能指标。
3. 再用 *"-DFPROFILE_USE=ON"* 二次编译；编译器利用统计信息优化分支等，从而提升可执行文件性能。

下图对比 MySQL 8.0.27 应用 PGO 前后、吞吐与并发的关系。

<img src="../media/image-20240829081620633.png" alt="image-20240829081620633" style="zoom:150%;" />

图 2-5. MySQL 8.0.27 中使用 PGO 前后的性能比较测试。

如图所示，低并发下 PGO 明显提升吞吐；但并发超过 150 后，总体吞吐与峰值反而下降。

PGO 是否只适合低并发，还是另有因素限制了它的收益？这涉及排队论与系统结构，后文会结合计算机基础继续分析。

## 2.3 可扩展性增强后线程池对 MySQL 的不利影响

在为 MySQL 8.0.27 打上多类可扩展性补丁后，需要再评估 Percona 线程池是否仍有帮助。下图是单机实例上 BenchmarkSQL TPC-C 的结果：深蓝线为启用线程池（大小 128），深红线为关闭；并发 50–2000，库规模 1000 warehouse。

<img src="../media/image-20240829081719592.png" alt="image-20240829081719592" style="zoom:150%;" />

图 2-6. 启用 Percona 线程池后吞吐明显低于关闭时。

如图所示，启用线程池后吞吐明显更差。即便不启用线程池，MySQL 8.0 相对 5.7 的可扩展性也已改善许多，因此线程池对这类 TPC-C 场景的额外收益有限；其自身开销也会体现在曲线上。

需要指出，在频繁建连、争用很重的场景里，Percona 线程池仍可能有价值。真正值得追问的是：究竟是什么让 MySQL 可扩展性有了如此大的改善？后续章节会继续讨论。

## 2.4 在 MySQL 8.0 中，TPC-C 吞吐量下降太快

长期 TPC-C 的常见要求是：至少跑满八小时，且任意两小时窗口内抖动小于 2% [14]。

我们在 MySQL 8.0.27 上用 BenchmarkSQL 做了长期 TPC-C。主要参数如下：

```
warehouses=1000
loadWorkers=100
terminals=200
warehouses-begin=1
warehouses-end=1000
//To run specified transactions per terminal- runMins must equal zero
runTxnsPerTerminal=0
//To run for specified minutes- runTxnsPerTerminal must equal zero
runMins=480
//Number of total transactions per minute
limitTxnsPerMin=0
//Set to true to run in 4.x compatible mode. Set to false to use the
//entire configured database evenly.
terminalWarehouseFixed=false
//The following five values must add up to 100
//The default percentages of 45, 43, 4, 4 & 4 match the TPC-C spec
newOrderWeight=45
paymentWeight=43
orderStatusWeight=4
deliveryWeight=4
stockLevelWeight=4
```

配置为 1000 个仓库、并发 200，且 `terminalWarehouseFixed=false`，即每个事务会换用不同 warehouse，从而扫到全库数据。

下图是长期测试中吞吐随时间的变化：下降幅度远超预期，接近 50%。

<img src="../media/image-degrade.png" alt="image-degrade" style="zoom:150%;" />

图 2-7. MySQL 8.0.27 的 BenchmarkSQL 测试期间暴露的性能下降。

该问题在 BenchmarkSQL 路径上复现，未必在所有 TPC-C 工具中出现。截至 MySQL 8.0.40，吞吐快速下滑仍未完全解决。后文会分析根因。

## 2.5 可重复读意外地优于读已提交

事务隔离是 ACID 中 “I” 的含义：在多事务并发改读时，在性能与结果可靠性、一致性、可预期性之间做权衡。常用级别包括读已提交（RC）、可重复读（RR）与串行化（Serializable）；InnoDB 默认 RR。

不同隔离级别的加锁策略不同，会改变查询在并发下的等待行为；按级别不同，查询可能要等其他会话释放锁后才能继续 [13]。一般认为级别越严，性能越差。MySQL 实际表现如何？

我们在串行化、RR、RC 上分别跑了 SysBench uniform（低冲突）与 pareto（高冲突）。pareto 场景死锁日志过多会干扰分析，故改源码抑制日志以保证对比公平；测试客户端也使用修改版以提高准确性。

uniform 结果如下：并发从 50 增到 800（按倍增）。低冲突下，低并发时三档隔离级别吞吐接近；超过 400 并发后，串行化明显掉速。

<img src="../media/image-20240829151823981.png" alt="image-20240829151823981" style="zoom:150%;" />

图 2-8. 不同隔离级别下低冲突的 SysBench 读写性能比较。

400 并发以下差异很小，是因为冲突少，锁策略影响被稀释。此时 RC 更受频繁获取 MVCC ReadView 的拖累，表现反而不如 RR。

再看 pareto 高冲突对比：

<img src="../media/image-20240829081950365.png" alt="image-20240829081950365" style="zoom:150%;" />

图 2-9. 不同隔离级别下高冲突的 SysBench 读写性能比较。

如图所示，冲突加重后，锁策略差异会直接体现在吞吐上；隔离越严通常越慢，符合直觉。

低冲突时，瓶颈更常落在 ReadView 获取：RC 每次读都要从全局活跃事务列表拷贝完整快照，而 RR 主要在事务开始时取一次。

小结：uniform 这类低冲突负载里，ReadView 开销可超过锁开销，于是 RR 优于 RC；pareto 这类高冲突负载里，锁开销主导，于是 RC 优于 RR。

## 2.6 组复制吞吐量低于半同步复制

组复制运行时会维护认证数据库，并定期清理过期认证信息以控制内存。清理需要拿全局锁存器，期间主节点执行会被暂时卡住。

半同步则要求从节点处理大量 relay log 事件；事件落盘后，从节点才向主节点回 ack。链路含网络、事件处理与刷盘，响应往往更长。

因此半同步下主节点必须等从节点“写盘后的确认”；组复制则在 Paxos 层达成共识后即可继续，不必等该层日志落盘。理论上组复制吞吐应更高。

两节点集群上，按并发对比组复制与半同步的 TPC-C 吞吐，见下图。

<img src="../media/image-20240829082024079.png" alt="image-20240829082024079" style="zoom:150%;" />

图 2-10. 组复制与半同步复制的性能比较。

如图所示：低并发半同步更好；高并发组复制曲线更稳，并在 250 并发达峰，但峰值仍低于半同步（半同步约在 100 并发达峰）。结果出人意料——问题出在哪？

关键在于组复制独有的认证数据库路径：大量内存分配/释放明显卡住吞吐上限。即便不做 Paxos 日志持久化，这一瓶颈也会抵消理论优势。认证库实现，是组复制性能的主要挑战之一。

## 2.7 修改后的组复制优于半同步复制

在解决 MySQL 8.0.32 可扩展性问题的过程中，组复制也做了大量增强。为验证效果，在同机两节点、各自独立 NVMe SSD、并用 NUMA 隔离的条件下，对比半同步与“带 Paxos 日志持久化的组复制”：主节点绑定 NUMA 0–2，从节点绑定 NUMA 3；除复制相关项外配置一致。

各并发下的吞吐对比如下。

<img src="../media/image-20240829082055680.png" alt="image-20240829082055680" style="zoom:150%;" />

图 2-11. 带有 Paxos 日志持久化的组复制与半同步复制的性能比较。

两者都有持久化：组复制持久化 Paxos 日志，半同步持久化 relay log。机制不同，前者吞吐明显更好。

Meta 实现了基于 Raft 的 MySQL 高可用；其开发者测试显示，Raft 改进版性能与半同步大致相当，见下图 [42]。

![](../media/d00bee2b821eb3d268b99fb6ea94f916.png)

图 2-12. 从 Meta 论文中借用的吞吐量比较。

理论上，组复制可用批处理方式做磁盘持久化，写盘阶段不必解析 binlog 事件，因而有望拿到更高吞吐。后文会继续讨论如何改组复制，以及原生半同步可扩展性受限的具体原因。

## 2.8 SysBench 无效果，TPC-C 表现良好

MySQL 8.0 中 lock-sys 优化的具体内容如下：

```c++
commit 1d259b87a63defa814e19a7534380cb43ee23c48
Author: Jakub Łopuszański <jakub.lopuszanski@oracle.com>
Date:   Wed Feb 5 14:12:22 2020 +0100
    WL#10314 - InnoDB: Lock-sys optimization: sharded lock_sys mutex

    The Lock-sys orchestrates access to tables and rows. Each table, and each row,
    can be thought of as a resource, and a transaction may request access right for
    a resource. As two transactions operating on a single resource can lead to
    problems if the two operations conflict with each other, Lock-sys remembers
    lists of already GRANTED lock requests and checks new requests for conflicts in
    which case they have to start WAITING for their turn.

    Lock-sys stores both GRANTED and WAITING lock requests in lists known as queues.
    To allow concurrent operations on these queues, we need a mechanism to latch
    these queues in safe and quick fashion.

    In the past a single latch protected access to all of these queues.
    This scaled poorly, and the management of queues become a bottleneck.

    In this WL, we introduce a more granular approach to latching.
```

这是 MySQL 8.0 通过打掉全局锁存器瓶颈、优化 InnoDB 锁调度来改善可扩展性的代表改动。先用 SysBench 读写验证效果：

<img src="../media/image-20240829082201268.png" alt="image-20240829082201268" style="zoom:150%;" />

图 2-13. lock-sys 优化前后的 SysBench 读写测试比较。

出乎意料，上线 lock-sys 优化后吞吐反而下降。为减少代码路径上 NUMA 兼容问题的干扰，把实例绑到 NUMA 节点 0（近似 SMP）再测：

<img src="../media/image-20240829082231096.png" alt="image-20240829082231096" style="zoom:150%;" />

图 2-14. SMP 下 lock-sys 优化前后的 SysBench 读写测试比较。

优化前后几乎无差别。说明原先结果被其他瓶颈“盖住”了；绑核后干扰变小，差距收敛，也说明该优化对标准 SysBench 读写帮助有限。

再用 BenchmarkSQL 做 TPC-C：

<img src="../media/image-20240829082255780.png" alt="image-20240829082255780" style="zoom:150%;" />

图 2-15. lock-sys 优化前后的 BenchmarkSQL 测试比较。

这次收益很清楚。于是问题变成：为何 SysBench 几乎看不出，BenchmarkSQL 却有效？两种工具差异及测试注意点，后文再系统讨论。

## 2.9 禁用 NUMA 对 MySQL 真的有益吗？

先看对主节点的影响。两台同规格 x86 机器跑 BenchmarkSQL 高压：一台 BIOS 关 NUMA，一台开 NUMA。吞吐随并发变化如下。

<img src="../media/image-20240829082344150.png" alt="image-20240829082344150" style="zoom:150%;" />

图 2-16. 通过在 BIOS 中禁用 NUMA 明显提高了 TPC-C 吞吐量。

如图所示，关 NUMA 后 TPC-C 吞吐明显上升。原因是 BIOS 关闭 NUMA 后内存分配路径对主库这类应用更有利。

那对从节点回放是否也有利？同一批机器上：关 NUMA 时无法再做 NUMA 绑定、可用全部内存；开 NUMA 时把从节点绑到节点 0。平衡回放速度对比如下。

<img src="../media/image-20240829082406694.png" alt="image-20240829082406694" style="zoom:150%;" />

图 2-17. 在 BIOS 中禁用 NUMA 前后的平衡回放速度比较。

关 NUMA 时平衡回放大约只有 570,000 tpmC，主因是从节点侧 NUMA 不友好问题尚未解决；开 NUMA 并绑单节点则可超过 810,000 tpmC。可见“BIOS 关 NUMA”对从库回放并不友好；若不先处理 NUMA 不友好问题，关 NUMA 反而会伤回放效率。第 10 章会专门分析这些点。

## 2.10 总结

本章列举了多类难分析的 MySQL 现象。后续解决路径从严密逻辑分析开始（见下一章），并要求同时吃透计算机基础与 MySQL 内部机制。计算机基础覆盖体系结构、数据结构、算法、操作系统、网络、编译、排队论、分布式理论等，详见第 4 章；MySQL 内部机制见第 5 章。

[下一页](Part2.md)
