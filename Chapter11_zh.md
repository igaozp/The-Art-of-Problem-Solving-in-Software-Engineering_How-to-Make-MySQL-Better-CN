# 第11章：性能调优

本章主要介绍在不修改 MySQL 源代码的情况下提升 MySQL 应用性能的方法。

## 11.1 PGO

Profile-guided optimization (PGO，配置文件引导优化）通常可以提高程序执行效率。下图展示了在解决 MySQL MVCC ReadView 可扩展性问题后，PGO 如何提升单机 MySQL 实例在不同并发级别下的吞吐量。

<img src="media/image-20240829113916829.png" alt="image-20240829113916829" style="zoom:150%;" />

图 11-1. 解决 MVCC ReadView 可扩展性问题后 PGO 的影响。

从图中可以明显看出，PGO 具有显著的影响。

对于使用 PGO 的 MySQL 8.0.27，在高并发条件下吞吐量会下降。具体细节如下图所示：

<img src="media/image-20240829113948830.png" alt="image-20240829113948830" style="zoom:150%;" />

图 11-2. MySQL 8.0.27 中使用 PGO 前后的性能对比测试。

上述测试结果表明，PGO 对 MySQL 的改进需要先解决可扩展性问题，才能充分发挥其潜力。需要注意的是，上述两组对比测试都是在主流的 NUMA 环境中进行的。当 MySQL 绑定到单个 NUMA 节点，创建 SMP 环境时，下图显示了使用 PGO 前后 TPC-C 吞吐量与并发级别的关系。

<img src="media/image-20240829114017540.png" alt="image-20240829114017540" style="zoom:150%;" />

图 11-3. SMP 环境下 MySQL 8.0.27 中使用 PGO 前后的性能对比测试。

从图中可以看出，PGO 在 SMP 环境中持续提升吞吐量，不会随着并发级别的增加而下降。下图比较了 PGO 在 NUMA 和 SMP 环境中的性能提升。

<img src="media/image-20240829114037360.png" alt="image-20240829114037360" style="zoom:150%;" />

图 11-4. PGO 优化在不同环境中的性能表现。

从图中可以明显看出，PGO 在 SMP 环境中实现了高达 30% 的最大性能提升，而在 NUMA 环境中，性能提升随着并发的增加而减少。这表明 PGO 在 SMP 环境中具有更大的潜力。

继续分析，研究了 PGO 在 Group Replication 集群环境中与单个 MySQL 实例相比的性能表现。下图描述了 Group Replication 的简化队列模型。

![](media/2eba13133f9eb007d89459cce5d4055b.png)

图 11-5. Group Replication 的简化队列模型。

由于网络部分无法被 PGO 优化，与单个 MySQL 实例相比，MySQL 主节点消耗的时间比例较低。根据 Amdahl 定律，PGO 带来的性能提升将不如单机 MySQL 实例明显。通常，随着网络延迟的增加，PGO 的改进效果趋于减弱。

下图比较了单机 MySQL 实例和 Group Replication 使用 PGO 的吞吐量提升。

<img src="media/image-20240829114100020.png" alt="image-20240829114100020" style="zoom:150%;" />

图 11-6. Group Replication 与单机 MySQL 中 PGO 的性能提升对比。

从图中可以看出，PGO 在 Group Replication 集群环境中的性能提升通常小于单机 MySQL 实例。

总之，PGO 可以总结如下：

1. 对于 MySQL 而言，PGO 是一项值得采用的优化，理论上可以全面提升性能，特别是在 SMP 环境中。
2. 在 NUMA 环境中，需要解决可扩展性问题才能从 PGO 中获得显著收益。
3. PGO 在 Group Replication 集群中的效果不如单机 MySQL 实例明显。

## 11.2 最小化网络交互

为了在不影响 TPC-C 测试正确性的前提下提高 MySQL 应用性能，修改 SQL 交互方式以减少客户端与 MySQL 之间的网络交互次数。这种优化可以间接提升性能。

以下示例使用优化的 tpcc-mysql 测试工具来确定在特定软硬件条件下 Group Replication 支持的最大吞吐量。

```
 [warehouse]: 200
[connection]: 200
    [rampup]: 0 (sec.)
   [measure]: 60 (sec.)
RAMP-UP TIME.(0 sec.)
MEASURING START.
  10, trx: 179372, 95%: 7.134, 99%: 9.097, max_rt: 16.857, 178944|16.256, 17941|5.395, 17932|25.313, 17943|14.180
  20, trx: 181215, 95%: 7.216, 99%: 9.228, max_rt: 20.587, 180703|433.405, 18123|4.086, 18122|29.192, 18124|4.922
  30, trx: 180596, 95%: 7.410, 99%: 9.463, max_rt: 16.930, 180110|16.705, 18061|3.146, 18056|25.658, 18060|7.392
  40, trx: 179032, 95%: 7.594, 99%: 10.575, max_rt: 18.047, 178513|16.862, 17906|3.861, 17900|27.482, 17904|5.724
  50, trx: 173856, 95%: 8.199, 99%: 12.705, max_rt: 31.777, 173412|30.990, 17388|9.501, 17388|37.472, 17385|5.538
  60, trx: 171950, 95%: 8.499, 99%: 13.721, max_rt: 31.584, 171546|31.808, 17200|9.185, 17188|35.098, 17202|13.781
STOPPING THREADS........................................................................................................................................................................................................
<Raw Results>
  [0] sc:481189 lt:584832  rt:0  fl:0 avg_rt: 5.3 (5)
  [1] sc:915412 lt:147816  rt:0  fl:0 avg_rt: 4.4 (5)
  [2] sc:106614 lt:5  rt:0  fl:0 avg_rt: 0.8 (5)
  [3] sc:106586 lt:0  rt:0  fl:0 avg_rt: 11.9 (80)
  [4] sc:106619 lt:0  rt:0  fl:0 avg_rt: 2.1 (20)
 in 60 sec.
<Raw Results2(sum ver.)>
  [0] sc:481239  lt:585014  rt:0  fl:0
  [1] sc:918008  lt:148245  rt:0  fl:0
  [2] sc:106622  lt:5  rt:0  fl:0
  [3] sc:106622  lt:0  rt:0  fl:0
  [4] sc:106628  lt:0  rt:0  fl:0
<Constraint Check> (all must be [OK])
 [transaction percentage]
        Payment: 43.41% (>=43.0%) [OK]
   Order-Status: 4.35% (>= 4.0%) [OK]
       Delivery: 4.35% (>= 4.0%) [OK]
    Stock-Level: 4.35% (>= 4.0%) [OK]
 [response time (at least 90% passed)]
      New-Order: 45.14%  [NG] *
        Payment: 86.10%  [NG] *
   Order-Status: 100.00%  [OK]
       Delivery: 100.00%  [OK]
    Stock-Level: 100.00%  [OK]
<TpmC>
                 1066021.000 TpmC
```

数据表明，在给定的软硬件测试环境中，Group Replication 实现了 106.6 万 tpmC 的峰值吞吐量，创下了新纪录。

这突显了在应用开发过程中减少与 MySQL 的网络交互次数可以显著提升性能。

## 11.3 使用先进的内存分配工具

MySQL 默认使用 jemalloc 内存分配工具，特别是 3.6 版本。测试数据表明，jemalloc 3.6 在性能方面并非最优。下图显示了在 x86 架构上 jemalloc 4.5 相比 jemalloc 3.6 的吞吐量提升。

<img src="media/image-20240829114526325.png" alt="image-20240829114526325" style="zoom:150%;" />

图 11-7. Jemalloc 4.5 在性能上优于 jemalloc 3.6。

从图中可以明显看出，jemalloc 4.5 在性能上优于 jemalloc 3.6。

对于 MySQL 从节点重放，请参考下图：

<img src="media/image-20240829114557565.png" alt="image-20240829114557565" style="zoom:150%;" />

图 11-8. 使用 jemalloc 4.5 获得更好的重放速度。

对优化后的 MySQL 重放进行测试显示，jemalloc 3.6 和 jemalloc 4.5 之间存在显著差异。该图展示了使用 jemalloc 4.5 后平衡重放速度的显著提升。

高效的内存分配工具不仅可以提升 MySQL 主服务器性能，还可以改善 MySQL 从节点的重放速度。

## 11.4 索引与性能

第 4.10.6 节提供了有关该主题的详细信息。

## 11.5 影响性能的关键 MySQL 参数

数据库具有众多可调整的系统参数，这些参数控制着内存分配、I/O 管理和日志记录等关键方面。这些参数的影响因配置而异，范围从显著到微小不等。这种可变性源于影响 MySQL 性能的众多因素，这使得性能结果的预测变得复杂。然而，可以识别导致异常吞吐量的特定因素。以下部分详细介绍了影响性能的常用参数。

### 11.5.1 "双一"对性能的影响

"双一"配置参数包括两个参数：*sync_binlog* 和 *innodb_flush_log_at_trx_commit*。

*sync_binlog* 参数描述如下：

<img src="media/b58fa402bb2b4e16e53b0bda44a0e30e.png" style="zoom:50%;" />

以下配置控制 MySQL Server 将二进制日志同步到磁盘的频率。

![](media/cd74a05babfcb9f75a746c9e98f2789b.png)

*innodb_flush_log_at_trx_commit* 参数描述如下：

<img src="media/c59aebec0401c408755e8d427d973899.png" style="zoom:50%;" />

以下配置在提交操作的严格 ACID 合规性与通过重新排列和批处理提交相关 I/O 操作可实现的更高性能之间取得平衡。调整默认值可以提高性能，但在发生崩溃时可能导致事务丢失。

![](media/a3767cf99551c83f2db7f46d52b340d3.png)

重要的是要注意，随着 MySQL 8.0 中可扩展性的提升以及 redo log 处理的优化，在高并发测试场景中，禁用"双一"设置对 MySQL 主服务器吞吐量的影响已经减弱。

首先使用 MySQL 8.0.27 进行对比测试。下图展示了启用和禁用"双一"时吞吐量与并发的关系。

<img src="media/image-20240829114945572.png" alt="image-20240829114945572" style="zoom:150%;" />

图 11-9. 禁用"双一"获得更好的性能。

根据该图，可以明显看出，在 MySQL 8.0.27 中，禁用"双一"显著提升了吞吐量。然而，这种性能提升随着并发的增加而减弱。

基于使用 PGO 的改进版 MySQL，禁用"双一"后的性能提升不太明显，如下图所示：

<img src="media/image-20240829115033197.png" alt="image-20240829115033197" style="zoom:150%;" />

图 11-10. 解决 MySQL 可扩展性瓶颈后，禁用"双一"带来的性能提升较少。

还可以观察到，性能提升的程度随吞吐量而变化，并非恒定。随着 MySQL 可扩展性瓶颈的解决，"双一"的影响逐渐减小。

由于诸如 *replica_preserve_commit_order* 排队等机制，"双一"对 MySQL 从节点重放的影响更加明显。下图比较了使用优化的 MySQL 版本在禁用"双一"前后 MySQL 从节点的平衡重放速度。

<img src="media/image-20240829115125819.png" alt="image-20240829115125819" style="zoom:150%;" />

图 11-11. 关闭"双一"获得更好的重放速度。

在相同条件下，禁用"双一"使 MySQL 从节点实现了 81 万 tpmC 的平衡重放速度，而启用"双一"将此速度降低到约 70 万 tpmC。因此，禁用"双一"显著提升了 MySQL 从节点的重放速度。

请注意，上述测试结果是在高性能 SSD 环境中获得的，SSD 硬件性能的差异可能导致不同的结果。

### 11.5.2 禁用 Binlog 的性能影响

启用二进制日志记录后，服务器会将所有修改数据的语句记录到二进制日志中，用于备份和复制目的。禁用 binlog 理论上可以消除队列等待时间，从而提高吞吐量并减少响应时间。

下图使用 BenchmarkSQL 比较了在相同并发级别下禁用 binlog 前后的 TPC-C 吞吐量。

<img src="media/image-20240829115210707.png" alt="image-20240829115210707" style="zoom:150%;" />

图 11-12. 禁用 binlog 获得更好的性能。

从图中可以明显看出，禁用 binlog 对性能有显著影响。

### 11.5.3 理解 Spin Delay 参数对性能的影响

当锁不可用时，竞争的线程必须等待锁。存在多种等待策略，其中无界自旋（或忙等待）是最简单的。在这种方法中，线程反复检查内存位置，直到值发生变化。这种方法会消耗资源，并可能在线程数超过 CPU 数量的超额订阅系统中导致抢占 [3]。

**自旋的优点：**

- 在未加载的系统中，自旋提供高性能和最小开销，因为它避免了操作系统协调，每次锁切换仅产生几个缓存未命中延迟。在具有共享缓存的片上多处理器上，锁获取和释放延迟可低至 150 纳秒。

**自旋的缺点：**

- 当系统加载时，自旋会降低性能，因为它减少了可用于有用工作的处理器数量，当可运行线程超过硬件上下文时会导致次优性能。此外，由于自旋不与操作系统协调，因此在调度决策中不考虑锁持有者或自旋者。

spin delay 参数有助于解决 NUMA（非统一内存访问）兼容性问题。通常，更好的 MySQL 可扩展性会减少 spin delay 参数的影响。相反，在可扩展性较差的情况下，spin delay 参数会明显影响性能，尽管其有效性有上限。

基于带有 lock-sys 优化的 MySQL 8.0.21 版本，下图比较了不同并发级别下的 TPC-C 吞吐量。

<img src="media/image-20240829115241352.png" alt="image-20240829115241352" style="zoom:150%;" />

图 11-13. 在 MySQL 8.0.21 中使用 spin delay 20 获得更好的性能。

该图显示，设置 *spin_delay=20* 显著缓解了 MySQL 8.0.21 带有 lock-sys 优化的可扩展性问题。然而，在高并发场景下，spin delay 参数会大幅增加 CPU 开销。根据论文"Locking Made Easy"，简单的自旋锁在低竞争下非常高效，但扩展性不佳 [63]。

在 MVCC ReadView 优化之前，使用 PGO 编译的 MySQL 版本中对 spin delay 参数的进一步分析详见下图。

<img src="media/image-20240829115301443.png" alt="image-20240829115301443" style="zoom:150%;" />

图 11-14. MVCC ReadView 优化前使用 spin delay 20 获得更好的性能。

从图中可以明显看出，当默认 *spin_delay* 参数设置为 6 时，吞吐量在 200 并发时急剧下降。然而，设置 *spin_delay=20* 会显著提升吞吐量。

让我们继续进行 MVCC ReadView 优化以进一步提升 MySQL 可扩展性。下图显示了 MVCC ReadView 优化与 *spin delay=20* 之间的吞吐量比较：

<img src="media/image-20240829115320081.png" alt="image-20240829115320081" style="zoom:150%;" />

图 11-15. MVCC ReadView 优化与 spin delay 20 的吞吐量比较。

从图中可以看出，虽然 spin delay 可以在一定程度上缓解可扩展性问题，但其效果明显不如采用 MVCC ReadView 优化带来的好处。

MVCC ReadView 优化后，再次测试了 *spin_delay* 的影响。具体结果见下图：

<img src="media/image-20240829115410814.png" alt="image-20240829115410814" style="zoom:150%;" />

图 11-16. 解决 MVCC ReadView 可扩展性问题后，spin delay 20 没有效果。

该图显示，在解决 MVCC ReadView 可扩展性问题后，*spin_delay* 参数的影响微乎其微。

### 11.5.4 Binlog 提交顺序对性能的影响

以下是对 *binlog_order_commits* 参数的简要介绍：

<img src="media/d4747b5b04d0887c1a49b974d187dc6f.png" style="zoom:50%;" />

当在复制源服务器上启用此变量（默认）时，对存储引擎的事务提交指令在单个线程上序列化，确保事务按照写入二进制日志的相同顺序提交。禁用此变量允许使用多个线程发出事务提交指令。与二进制日志组提交结合使用，这可以防止单个事务的提交率成为吞吐量的瓶颈，因此可能产生性能提升 [13]。

基于 MySQL 8.0.27 版本，检查启用 *binlog_order_commit* 参数是否影响 TPC-C 吞吐量。具体比较见下图：

<img src="media/image-20240829115454755.png" alt="image-20240829115454755" style="zoom:150%;" />

图 11-17. binlog 组提交优化前，关闭 binlog_order_commits 获得更好的性能。

从图中可以看出，禁用 *binlog_order_commits* 在不同并发级别下显著提升了 TPC-C 吞吐量。

下图基于使用 PGO 设置优化的改进版 MySQL。禁用 *binlog_order_commits* 对 TPC-C 吞吐量的提升不太明显。解决了 binlog 组提交引起的惊群问题等问题提高了 MySQL 效率，并缓解了启用 *binlog_order_commits* 时观察到的性能下降。

<img src="media/image-20240901095750255.png" alt="image-20240901095750255" style="zoom:150%;" />

图 11-18. binlog 组提交优化后，关闭 binlog_order_commits 带来的性能提升较少。

MySQL 默认启用 *binlog_order_commits*。值得注意的是，禁用 *binlog_order_commits* 会影响 MySQL 的克隆功能。

### 11.5.5 全面理解 Performance Schema 对性能的影响

Performance Schema 通过进行实际测量而非猜测来帮助 DBA 调优性能。不幸的是，在 NUMA 环境中，启用 Performance Schema 会影响 MySQL 的性能，特别是影响 MySQL 从节点重放。下图使用 BenchmarkSQL 比较了不同并发级别下的 TPC-C 吞吐量以分析这种影响。

<img src="media/image-20240829115606145.png" alt="image-20240829115606145" style="zoom:150%;" />

图 11-19. 关闭 performance_schema 获得更好的性能。

从图中可以看出，Performance Schema 对 MySQL 主节点性能的影响相对较小，用户通常可以接受。然而，启用 Performance Schema 会显著影响 MySQL 从节点的重放速度。下图显示了启用 Performance Schema 后 Group Replication 重放队列大小的变化。

![](media/2e14f81ddb33dc7a1db1d7f41e69dcdd.png)

图 11-20. 启用 Performance Schema 后 Group Replication 中的重放队列大小增加。

从图中可以观察到重放队列大小正在增加，表明 MySQL 从节点无法跟上 MySQL 主节点的处理速度。在 MySQL 主节点上执行后的吞吐量结果如下：

![](media/42665a7506959bdefa1b451ce564a261.png)

MySQL 主节点完成执行后，以下是在 MySQL 从节点重放过程中捕获的性能截图：

![](media/9b468bff3f312fa70821da3a0a117f70.png)

图 11-21. 与 Performance Schema 相关的重大瓶颈。

从绿色高亮框中可以明显看出，存在与 Performance Schema 相关的重大瓶颈。

总结上述测试结果：当 MySQL 主节点实现 50 万 tpmC 的吞吐量时，启用 Performance Schema 的 MySQL 从节点显示出明显的瓶颈。此外，MySQL 从节点在重放过程中无法跟上 MySQL 主节点的速度。

现在，禁用 Performance Schema，进行相同的并发测试。具体结果见下图：

![](media/74a2f4f519d0991c114aabf3318369b2.png)

图 11-22. 禁用 Performance Schema 后 Group Replication 中的重放队列大小不会增加。

该图表明，等待重放的事务数量并未随时间显著增加。MySQL 主节点完成执行后的吞吐量结果如下：

![](media/468b1638646ec6ce6ae997ec6e52f152.png)

当 MySQL 主节点测试完成时，MySQL 从节点几乎同时完成重放。这表明在主流 NUMA 环境中，Performance Schema 对 MySQL 从节点重放极为不利。Performance Schema 的 NUMA 兼容性问题尚未解决，将成为未来优化的重点。

### 11.5.6 Doublewrite 对性能的影响

以下是对 *innodb_doublewrite* 参数的简要介绍：

<img src="media/5dc2c211205677ed7ae1528cd1911685.png" style="zoom:50%;" />

doublewrite 缓冲区是一个存储区域，InnoDB 在将页面从缓冲池提交到数据文件之前将其写入该区域。此机制通过在发生故障时提供可靠的页面副本来确保崩溃恢复期间的数据完整性。然而，由于默认的 InnoDB 页面大小为 16 KB，而文件系统只能保证最多 4 KB 的原子性，因此可能只有部分页面可恢复。启用 doublewrite 有助于 InnoDB 通过两次写入来恢复完整页面。

使用带有 PGO 的改进版 MySQL 8.0.27 评估启用 doublewrite 对性能的影响。下图显示了在不同并发级别下禁用 doublewrite 对 TPC-C 吞吐量的影响。

<img src="media/image-20240829115642934.png" alt="image-20240829115642934" style="zoom:150%;" />

图 11-23. 禁用 doublewrite 获得更高的峰值吞吐量。

从图中可以明显看出，禁用 doublewrite 参数可以提高峰值吞吐量。虽然广泛的测试和统计分析表明该参数对整体性能的影响有限，但它确实有助于提高峰值吞吐量。然而，禁用 doublewrite 可能会在数据库崩溃时损害恢复安全性，因此应谨慎使用。

### 11.5.7 Binlog Row Image 格式对性能的影响

以下是对 *binlog_row_image* 参数的简要介绍：

<img src="media/90962de7267c4d56e20456f01796ac29.png" style="zoom:50%;" />

对于 MySQL 基于行的复制，此变量控制如何在二进制日志中记录行映像 [13]。每个行更改事件由两个映像组成：一个"前"映像，表示更改前行的状态，以及一个"后"映像，反映更新后的状态。通常，MySQL 会为两个映像记录所有列。然而，可以只记录必要的列，这可以减少磁盘、内存和网络使用。

基于使用 PGO 的改进版 MySQL 8.0.27，分析了 *binlog_row_image* 参数在不同并发级别下对 TPC-C 吞吐量的影响。见下图：

<img src="media/image-20240829115703081.png" alt="image-20240829115703081" style="zoom:150%;" />

图 11-24. 使用最小日志记录获得改进的吞吐量。

从图中可以观察到，使用 *binlog_row_image=minimal* 为 MySQL 主节点提供了适度的吞吐量提升，但增幅并不显著。关于 MySQL 从节点重放，具体细节可以在第 10.3.7 节中找到。

### 11.5.8 理解 Binlog 压缩设置对性能的影响

以下是对 *binlog_transaction_compression* 参数的简要介绍：

<img src="media/f621d8dd39147a621753b36d643cf42f.png" style="zoom:50%;" />

启用二进制日志事务压缩后，事务有效负载会被压缩并作为单个事件写入二进制日志。这些压缩的有效负载在复制流和中继日志中保持压缩状态，减少了存储需求并节省了事务发起者和接收者的网络带宽 [13]。

在同一数据中心内的 Group Replication 集群中，为 MySQL 主节点和从节点启用 *binlog_transaction_compression*，让我们分析二进制日志压缩对 TPC-C 吞吐量的影响。详细结果请参考下图：

<img src="media/image-20240829115724267.png" alt="image-20240829115724267" style="zoom:150%;" />

图 11-25. 在 LAN 环境中 Binlog 压缩可能对性能产生负面影响。

从图中可以明显看出，启用 binlog 压缩会导致 Group Replication 集群的吞吐量显著下降，特别是在高并发情况下。当压缩的网络收益被所涉及的计算成本所抵消时，就会出现这种下降。

为了研究增加网络延迟是否可以改善压缩后的吞吐量，使用了 10ms 的模拟网络延迟。binlog 压缩前后 TPC-C 吞吐量与并发级别的比较如下图所示。

<img src="media/image-20240901103231333.png" alt="image-20240901103231333" style="zoom:150%;" />

图 11-26. 在 WAN 环境中 Binlog 压缩不影响性能。

从图中可以明显看出，在 10ms 网络延迟的情况下，压缩后的吞吐量与压缩前相似。这表明增加的网络延迟可以帮助缓解与 MySQL 压缩相关的性能下降。

用户应根据其具体需求权衡存储成本降低与性能提升之间的取舍。

### 11.5.9 理解 InnoDB Buffer Pool 大小对性能的影响

以下是对 *innodb_buffer_pool_size* 参数的简要介绍：

<img src="media/c649a1a7c43cffd86e55f28aeb8ff607.png" style="zoom: 40%;" />

较大的缓冲池在内存中保留更多数据，减少了页面驱逐、I/O 操作和缓冲池内的竞争 [13]。

基于使用 PGO 的改进版 MySQL 8.0.27，分析了 *innodb_buffer_pool_size* 参数在不同并发级别下对 TPC-C 吞吐量的影响。仓库数量为 1000 时的结果如下图所示。

<img src="media/image-20240901104239714.png" alt="image-20240901104239714" style="zoom:150%;" />

图 11-27. innodb_buffer_pool_size：最佳大小是性能的关键，而不仅仅是更大。

从图中可以明显看出，增加缓冲池大小并不总是导致更高的吞吐量。对于 1000 个仓库，在测试中占用约 92GB 的 I/O 空间，将 *innodb_buffer_pool_size* 设置为 92GB 比设置为 192GB 性能更好。

值得注意的是，*innodb_buffer_pool_size* 参数非常适合使用 AI 进行动态自适应调优。

## 11.6 总结

本章重点介绍了在不修改 MySQL 源代码的情况下优化 MySQL 应用性能的方法。关键方法包括使用 PGO、最小化网络交互、采用先进的内存分配工具以及改进索引和参数配置。

这些优化方法的有效性取决于 MySQL 版本、配置参数、硬件环境和具体应用特性。虽然一个参数在某个场景中可能显著影响性能，但其在另一个场景中的有效性可能会降低。这种可变性源于影响性能的多个队列和潜在瓶颈之间的复杂交互，使性能测试特别具有挑战性。

[下一章](Part5.md)
