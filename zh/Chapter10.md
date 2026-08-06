# 第10章：MySQL 从库回放的重大改进

## 10.1 MySQL 从库回放机制

MySQL 支持主从复制（primary/secondary），常见做法是日志传输（log shipping）：事务先在主库执行，主库把操作写入日志并发送给从库；从库按序回放，使状态与主库一致 [16]。

这里说的回放，指按提交顺序回放事务；提交前互不冲突的事务可以并行。

异步复制、半同步与组复制（Group Replication）的回放流程本质相同。从库回放必须正确、支持一致读，并且要足够快，高可用才站得住——从库回放质量直接决定集群表现。

### 10.1.1 确保回放正确性

正确回放依赖事务间依赖关系：有冲突时必须定先后，依赖来自 binlog/relay log 中的事务顺序。

此外还要保证幂等（idempotence），崩溃恢复等场景尤其重要，避免重复应用带来副作用。

基于 GTID 保证幂等的关键逻辑如下：

```c++
  const bool skip_transaction = is_already_logged_transaction(thd);
  if (gtid_next_list == nullptr) {
    if (skip_transaction) {
      skip_statement(thd);
      return GTID_STATEMENT_SKIP;
    }
    return GTID_STATEMENT_EXECUTE;
  } else {
```

*is_already_logged_transaction* 函数的具体细节如下所述：

```c++
/**
  Check if current transaction should be skipped, that is, if GTID_NEXT
  was already logged.
  @param  thd    The calling thread.
  @retval true   Transaction was already logged.
  @retval false  Transaction must be executed.
*/
bool is_already_logged_transaction(const THD *thd);
```

本质上，它检查 GTID 事务是否已经被执行，如果已执行则跳过它。

除了在不同事务之间建立回放关系和维护幂等性之外，MySQL 事务回放还需要支持重试过程。在极端情况下，由于冲突的锁策略，回放有时会导致死锁。当发生这种情况时，被终止的事务需要再次回放，这涉及到重试过程的执行。

### 10.1.2 支持一致性读取

为了确保从 MySQL 从库读取时的一致性，请考虑以下几点：

1. **优化回放速度**：加速 MySQL 从库上的事务回放，以减少落后于主库的延迟。
2. **保持提交顺序**：为了维持因果关系，MySQL 从库按照事务写入 relay log 的顺序提交事务。这通过设置参数 *replica_preserve_commit_order=on* 来控制。
3. **读己之所写一致性**：为了满足"读己之所写"（read your writes）一致性要求，检查最近回放的事务的 GTID 是否已达到指定的 GTID 值。如果没有，则等待直到达到。这确保了读取能够反映最近的写入。

### 10.1.3 MySQL 从库回放的效率

MySQL 从库回放的执行过程与主库不同。虽然从库回放专注于执行事务事件和提交操作（主要处理写操作），但主库处理事务 SQL 的读写操作。

下图比较了标准 TPC-C 测试期间 MySQL 主库和从库的 CPU 消耗。

<img src="../media/image-20240829112229366.png" alt="image-20240829112229366" style="zoom:150%;" />

图 10-1. 从库的 CPU 消耗明显低于主库。

基于该图，可以得出以下结论：

1. **CPU 消耗比较**：

   无论并发级别如何，MySQL 从库回放的 CPU 消耗始终明显低于 MySQL 主库。这是由于：

   - MySQL 从库回放只涉及写操作，避免了读操作。
   - 基于事件的回放通常提高了执行效率。

2. **并发影响**：

   主库和从库之间的 CPU 消耗差异在低并发时较小，但在高并发时变得更加明显。

上述统计数据基于标准 TPC-C 测试结果，不包括大事务。下图比较了 TPC-C 数据加载过程中的时间消耗。

<img src="../media/image-20240829112250200.png" alt="image-20240829112250200" style="zoom:150%;" />

图 10-2. 处理大事务时，从库的 CPU 使用率远低于主库。

如图所示，MySQL 主库上的 CPU 消耗明显高于 MySQL 从库。这表明对于数据加载，包括处理大事务，MySQL 从库显示出非常高的执行效率。相比之下，MySQL 主库在处理大事务方面可能还有进一步的优化机会。

接下来 MySQL 主库加载 1000 个仓库的 TPC-C 数据所需的时间。具体统计结果见下图：

<img src="../media/image-20240829112326190.png" alt="image-20240829112326190" style="zoom:150%;" />

图 10-3. 将 MySQL 主库绑定到单个 NUMA 节点可提高 BenchmarkSQL 数据加载速度。

由此可以推断，在 MySQL 主库处理大事务时，改进 NUMA 兼容性方面存在相当大的潜力。

基于一系列对比测试，可以得出以下结论：

1. MySQL 从库回放在处理用户事务方面的效率明显超过 MySQL 主库。
2. MySQL 主库在处理大事务时显示出 NUMA 兼容性问题。

### 10.1.4 MySQL 从库的理论最大回放速度

论文 "Scalable Replay-Based Replication For Fast Databases" 说明了回放障碍（replay barrier）[18]：

一个重大挑战是从库通过按串行顺序回放数据库日志来确保与主库的一致性。这种回放难以并发执行，因此从库性能可能无法随主库性能扩展。

MySQL 从库上的事务回放过程包括两个主要阶段：

1. **事务事件的回放**：这涉及执行从主库接收的事务事件。
2. **提交**：此阶段通过提交更改来完成事务。

更多详情见下图。

![](../media/3efd7f636001d03ebf6fa17c39ae43b5.png)

图 10-4. MySQL 从库上的事务回放过程。

为了确保读取一致性，MySQL 从库上的提交操作必须顺序执行，这一点很关键。具体来说：

- **并行回放**：如果并发事务之间没有冲突，它们可以并行回放。
- **顺序提交**：在提交阶段，事务必须按照它们最初执行的顺序提交。

在性能方面，MySQL 从库回放的队列模型可以简化为下图所示。

![](../media/4a9ca333ba0b6b18acb7572285e92c26.png)

图 10-5. MySQL 从库回放的队列模型。

在 MySQL 从库回放中，多队列阶段——例如 relay log 刷新、事务事件回放（包括读取、解析和事件排队）以及提交操作——限制了理论上的最大回放速度。这些串行化的过程对回放速度的提升造成了固有的限制。

## 10.2 MySQL 回放慢的根本原因分析

### 10.2.1 MySQL 从库回放的队列模型

这里进一步细化了 MySQL 从库回放的队列模型，如下：

![](../media/4e3a6edca9e2bdafe7dd69fbdf76c8db.png)

图 10-6. MySQL 从库回放的更详细队列模型。

MySQL 从库回放采用分阶段模型，其中不同的处理阶段通过队列相互连接。无论是异步复制、半同步复制还是组复制，都一致使用此模型。

最初，MySQL 从库将接收到的事务事件写入 relay log。然后 SQL 线程从 relay log 读取这些事件，并将它们分发到各自的 worker 队列。

为什么写入和读取 relay log 事件的这些过程是必要的，而不是被跳过？如果 MySQL 从库无法跟上回放事件的速度，队列中的不平衡会导致队列大小变大。使用磁盘上的 relay log 事件将文件作为队列，避免了内存队列可能出现的内存不足（Out-Of-Memory，OOM）问题。

SQL 线程的速度需要灵活；如果太慢，事件无法及时回放；如果太快，未回放的事务可能在 worker 队列中累积，增加内存使用开销。因此，在需要时可能需要流量控制。

在 worker 队列中，MySQL 对队列大小施加限制，这不利于高效回放大事务。现代服务器具有更大的内存容量，将 worker 队列大小限制为 16,384 通常不足以满足回放需求。为了跟上 MySQL 主库的速度，增加 worker 队列大小很关键；否则，队列会很快填满，导致 SQL 线程不必要地等待。

在 MySQL 从库回放期间的提交操作方面，按照进入 relay log 的顺序提交为一致性读操作奠定了基础。这从根本上区分了 MySQL 从库回放与 MySQL 主库上用户 SQL 的执行：主库需要解析 SQL、生成执行计划并处理一系列读写操作，这会产生大量 CPU 开销，但不受上述队列模型的约束。相比之下，MySQL 从库回放只需要基于事务事件进行回放，CPU 开销较低，但受到上述队列模型的限制。

### 10.2.2 瓶颈分析

在 MySQL 异步复制中，从库有两个线程：IO 线程从主库读取日志条目并将其存储在本地磁盘上，而 SQL 线程从本地磁盘读取日志并回放事件。单个线程按顺序解析和执行日志事件。当解析行事件时，它包括操作类型和行镜像，这些以原始字节形式存储。SQL 线程必须将这些字节"解包"到字段中，在执行操作之前需要了解表模式。这种在将日志事件发送到多个线程进行回放之前的解包和检查是昂贵的，如果由单个线程处理可能成为瓶颈 [18]。具体而言，解包行镜像会消耗 SQL 线程的大量 CPU 资源，严重限制了 MySQL 从库回放性能。

例如，在组复制从库回放期间，捕获到的主要瓶颈之一是 SQL 线程，具体显示在 *rpl_rca_sql-0* 线程中，见下图。

![](../media/3b54c6545f0c49be73fd3afd9c6ffe79.gif)

图 10-7. 组复制从库回放中 CPU 消耗最高的两个线程。

捕获 SQL 线程的性能数据并生成火焰图可以深入了解 CPU 资源的消耗位置。具体见下图。

![](../media/6439fb2f520bc6d99229517ca9bdbe6e.png)

图 10-8. SQL 线程火焰图中识别的瓶颈。

如图所示，SQL 线程遇到两个主要瓶颈。一个重要的瓶颈是事件解析过程（如图中的 *binlog_event_deserialize* 函数所示），另一个主要瓶颈是 *malloc* 调用。

切换到 jemalloc 4.5 作为内存分配工具后，不仅 MySQL 从库回放速度提高了，SQL 线程本身的 CPU 开销也降低了。详情见下图。

![](../media/c7ef66e1ace9b544ba85d64f3f4ed1c4.gif)

图 10-9. 通过改进的 jemalloc 减少了 SQL 线程的 CPU 开销。

继续查看下面的新火焰图，很明显 SQL 线程在内存分配方面的开销显著减少。

![](../media/f01e9f4fd7eafca9b8a6eceeb6aec383.png)

图 10-10. SQL 线程的内存瓶颈已解决。

如图所示，*binlog_event_deserialize* 约占工作负载的 45%。事件解析的速度直接影响最大回放速度。为了解决这个瓶颈，在多个线程之间并行化事件解析可以明显提高性能。

一旦解决了 *binlog_event_deserialize* 瓶颈，下一步是识别影响回放速度的其他潜在瓶颈。查看 worker 队列的相关源代码可以发现，其大小限制为 16,384。

```c++
/*
  MTS load-balancing parameter.
  Max length of one MTS Worker queue. The value also determines the size
  of Relay_log_info::gaq (see @c slave_start_workers()).
  It can be set to any value in [1, ULONG_MAX - 1] range.
*/
const ulong mts_slave_worker_queue_len_max = 16384;
```

很明显，worker 队列最多可以容纳 16,384 个事件。虽然这个容量通常足以应对标准的事务量，但对于较大的事务来说就不够了。广泛的测试表明，这种限制明显限制了回放速度，特别是对于加载 TPC-C 仓库数据等任务。以下代码演示了一个满的 worker 队列可以阻塞 SQL 线程的执行。

```c++
  // possible WQ overfill
  while (worker->running_status == Slave_worker::RUNNING && !thd->killed &&
         (ret = worker->jobs.en_queue(job_item)) ==
             Slave_jobs_queue::error_result) {
    thd->ENTER_COND(&worker->jobs_cond, &worker->jobs_lock,
                    &stage_replica_waiting_worker_queue, &old_stage);
    worker->jobs.overfill = true;
    worker->jobs.waited_overfill++;
    rli->mts_wq_overfill_cnt++;
    // wait if worker queue is full
    mysql_cond_wait(&worker->jobs_cond, &worker->jobs_lock);
    mysql_mutex_unlock(&worker->jobs_lock);
    thd->EXIT_COND(&old_stage);
    mysql_mutex_lock(&worker->jobs_lock);
  }
```

大事务可以快速填满 worker 队列，导致 SQL 线程保持空闲状态，失去分发其他事务的机会。

SQL 线程不仅在 worker 队列满时阻塞，而且如果待回放的事务依赖于尚未完成回放的先前事务，它也会等待。详情请参见下面的具体代码：

```c++
bool Mts_submode_logical_clock::wait_for_last_committed_trx(
    Relay_log_info *rli, longlong last_committed_arg) {
  THD *thd = rli->info_thd;
  ...
  if ((!rli->info_thd->killed && !is_error) &&
      !clock_leq(last_committed_arg, get_lwm_timestamp(rli, true))) {
    PSI_stage_info old_stage;
    struct timespec ts[2];
    set_timespec_nsec(&ts[0], 0);
    assert(rli->gaq->get_length() >= 2);  // there's someone to wait
    thd->ENTER_COND(&rli->logical_clock_cond, &rli->mts_gaq_LOCK,
                    &stage_worker_waiting_for_commit_parent, &old_stage);
    do {
      // wait if LWM is less than last committed
      mysql_cond_wait(&rli->logical_clock_cond, &rli->mts_gaq_LOCK);
    } while ((!rli->info_thd->killed && !is_error) &&
             !clock_leq(last_committed_arg, estimate_lwm_timestamp()));
    min_waited_timestamp.store(SEQ_UNINIT);  // reset waiting flag
    mysql_mutex_unlock(&rli->mts_gaq_LOCK);
    thd->EXIT_COND(&old_stage);
    set_timespec_nsec(&ts[1], 0);
    rli->mts_total_wait_overlap += diff_timespec(&ts[1], &ts[0]);
  } else {
    min_waited_timestamp.store(SEQ_UNINIT);
    mysql_mutex_unlock(&rli->mts_gaq_LOCK);
  }
  return rli->info_thd->killed || is_error;
}
```

该代码说明了一种机制，如果记录的低水位标记（low-water-mark，LWM）——表示一个事务及其之前的所有事务都已提交——小于正在回放的事务的 last committed 值，则 SQL 线程会等待。在 MySQL 中，是 SQL 线程在等待，而不是 worker 线程。这种等待机制明显限制了回放速度。

最后，下面看与 NUMA 环境中 MySQL 从库回放相关的问题。下图给出 MySQL 从库回放的测试结果：

<img src="../media/image-20240829112700824.png" alt="image-20240829112700824" style="zoom:150%;" />

图 10-11. 将 MySQL 从库绑定到单个 NUMA 节点可提高回放速度。

利用所有 NUMA 节点可以实现约 245,000 tpmC 的平衡回放速度。这意味着如果 MySQL 主库的吞吐量超过此值，MySQL 从库通常无法跟上。

相比之下，当 MySQL 从库绑定到 NUMA 节点 0 时，平衡回放速度增加到 352,000 tpmC，提高了 43.7%。这种改进归因于在 MySQL 从库回放期间观察到的显著全局锁存器（latch）争用。在 NUMA 环境中，这种争用导致不同 NUMA 节点之间频繁的缓存迁移，降低了回放效率。将 MySQL 从库绑定到 NUMA 节点 0 可以通过避免 NUMA 节点之间的缓存迁移来缓解这种低效率，从而实现显著的性能提升。

## 10.3 MySQL 从库回放优化

### 10.3.1 事务事件解析的并行化

这里实现了一个拆分 SQL 线程的新模型。

![](../media/8276dc5ea73285406fa620d891bbc23a.png)

图 10-12. 拆分 SQL 线程的新模型。

在 MySQL 中，用于从库回放的 SQL 线程被分为六个线程：一个用于读取事件，四个用于解码事件，一个用于处理调度（调度线程）。这些线程通过队列连接。为了管理过多的内存使用，当队列大小变得太大时，对事件读取线程应用流量控制。

下面是 MySQL 从库运行过程的 *'top'* 截图。

![](../media/dccad19c5a07ffcd2477a9030d9337e7.gif)

图 10-13. SQL 线程在 *'top'* 显示中显示为六个独立线程。

通过拆分 SQL 线程，解决了单个线程处理能力有限的问题，有效消除了事件解析的瓶颈。

### 10.3.2 改进大事务的并行回放

对于大事务回放，主要问题在于 worker 队列太小，导致 SQL 线程频繁等待。

为此，可以设置更大的队列大小，如下面的代码所示。

```c++
/*
  MTS load-ballancing parameter.
  Max length of one MTS Worker queue. The value also determines the size
  of Relay_log_info::gaq (see @c slave_start_workers()).
  It can be set to any value in [1, ULONG_MAX - 1] range.
*/
const ulong mts_slave_worker_queue_len_max = 16384 << 6;
```

将 worker 队列大小增加 64 倍至 1,048,576 个事务事件，可以容纳具有许多事件的大事务，使 MySQL 从库能够充分利用其容量回放这些大事务。然而，这种增加可能会引起对内存使用的担忧。下图说明了具有不同 worker 队列大小和不同数量 worker 的 MySQL 从库进程的内存消耗。横轴表示 worker 的数量，纵轴显示 MySQL 从库进程的实际内存使用量。

<img src="../media/image-20240829112906641.png" alt="image-20240829112906641" style="zoom:150%;" />

图 10-14. 具有不同 worker 队列大小和不同数量 worker 的 MySQL 从库进程的内存消耗。

如图所示，MySQL 从库的实际内存消耗同时受到 *worker_queue_len_max* 和 worker 数量的影响。在高端机器上，额外的内存使用可能不是主要问题。然而，在低端机器上，减少 worker 数量可以帮助更有效地管理内存消耗。请记住，不同的硬件和软件系统可能会产生不同的结果，因此上述数据仅供参考。具体环境应进行定制测试。

目前，MySQL 不提供调整 *worker_queue_len_max* 参数的配置选项。尽管如此，在高端机器上，此参数在提高大事务的回放速度方面起着关键作用。

### 10.3.3 优化调度线程机制

如果在 MySQL 中时间上接近发生的两个事务修改相同的数据，它们可能在从库回放期间产生依赖关系。具体而言，后一个事务的 *last_committed* 值可能与前一个事务的 *sequence_number* 匹配。在 MySQL 从库回放期间，如果后一个事务依赖于尚未完成回放的前一个事务，调度线程将阻塞自己。这种方法效率低下，因为 worker 回放线程理想情况下应该处理这种等待，而不是调度线程。调度线程本身有很多其他任务要处理，并且无法保证后续事务不能及时回放。

为了更好地理解这个问题，请考虑以下情况：

![](../media/9aa41ee1e12bb8a72dd240dc42b44ba5.gif)

图 10-15. *sequence_number* 和 *last_committed* 的典型示例。

假设 MySQL 从库已经完成了事务 12756（*LWM*=12756），当调度线程调度事务 12759（图中绿色框中突出显示）时，它发现依赖的事务 12757 尚未完成回放。结果，调度线程阻塞自己，直到事务 12757 完成。同时，事务 12760、12761、12762、12763 和 12764 可以分配给 worker 进行回放。对事务 12757 的等待可以由 worker 回放线程处理，从而使调度线程可以继续执行其他任务。这提供了优化的机会。

下图说明了在各种并发级别下使用 BenchmarkSQL 进行的 TPC-C 测试，以评估 MySQL 从库是否能匹配 MySQL 主库的速度。横轴表示并发级别，纵轴显示 tpmC 值。浅灰色区域表示 MySQL 从库跟上了 MySQL 主库，而深灰色区域表示没有跟上。

<img src="../media/image-20240829113049624.png" alt="image-20240829113049624" style="zoom:150%;" />

图 10-16. 如果 MySQL 主库的处理速度超过平衡回放速度，从库可能跟不上。

如图所示，当并发达到 70 时，MySQL 从库落后于 MySQL 主库的处理速度。这种低效率主要是由于调度线程不必要的等待，这破坏了正常的回放过程。只要吞吐量低于 245,439 tpmC，MySQL 从库就能跟上主库。然而，一旦超过这个阈值，MySQL 从库就难以匹配主库的速度。本质上，MySQL 从库回放无法有效支持高吞吐量。

为此，允许 worker 独立处理对 **last_committed** 事务的等待可以完全释放调度线程。这种调整为未来改进 MySQL 从库回放性能提供了坚实的基础。

### 10.3.4 内存分配工具对回放性能的影响

利用先进的内存分配工具可以提高 MySQL 从库的回放速度。下图给出两个 jemalloc 版本在 x86 架构下对 MySQL 从库回放性能的影响。

<img src="../media/image-20240829113113252.png" alt="image-20240829113113252" style="zoom:150%;" />

图 10-17. 使用 jemalloc 4.5 实现更好的回放速度。

如图所示，使用 jemalloc 4.5 明显提高了 MySQL 从库的回放速度。在相同条件下，jemalloc 3.6 实现的平衡回放速度刚刚超过 710,000 tpmC，而 jemalloc 4.5 达到 810,000 tpmC，性能提高了 14%。

### 10.3.5 解决 NUMA 环境中的不友好问题

在 BIOS 级别禁用 NUMA 的场景中，评估了 MySQL 从库回放过程的效率。详情见下图。

<img src="../media/image-20240829113134905.png" alt="image-20240829113134905" style="zoom:150%;" />

图 10-18. 在 BIOS 中禁用 NUMA 前后平衡回放速度的比较。

在 BIOS 中禁用 NUMA 导致回放速度明显下降。虽然此更改会影响内存分配方法，但不会改变底层的 NUMA 硬件架构。

是否禁用 NUMA 的决定应基于全面的分析。在 x86 架构上的测试表明，禁用 NUMA 会对 MySQL 从库回放产生负面影响，但可能有利于 MySQL 主库操作。进一步调查 NUMA 与 MySQL 从库回放的不兼容性可能会揭示 MySQL 本身更深层次的问题。

基于 MySQL 的 Performance Schema，收集了有关锁存器操作的统计信息。结果见下图：

![](../media/08fb8a25d9542121b1e5b5ccb3cbdf6f.gif)

图 10-19. pending_jobs_lock 的瓶颈已在 Performance Schema 中暴露。

如图所示，在 NUMA 环境中，*Relay_log_info::pending_jobs_lock* 在与回放相关的重要性方面排名第五。将 MySQL 从库实例绑定到 NUMA 节点 0，测试评估在 SMP 环境下 Performance Schema 统计信息是否发生变化。具体测试结果见下图：

![](../media/28708a58742f84033b84ff551cda3b1b.gif)

图 10-20. 在 SMP 环境中 pending_jobs_lock 的瓶颈已得到缓解。

如图所示，*Relay_log_info::pending_jobs_lock* 已移至第八位，表明在绑定到 NUMA 节点 0 的场景下，与 *Relay_log_info::pending_jobs_lock* 相关的争用已得到缓解。

在 MySQL 源代码中，*pending_jobs_lock* 的相关代码在两个地方被频繁调用。第一次出现在 *remove_item_from_jobs* 函数中，如下面的代码片段所示：

```c++
/**
  Remove a job item from the given workers job queue. It also updates related
  status.
  param[in] job_item The job item will be removed
  param[in] worker   The worker which job_item belongs to.
  param[in] rli      slave's relay log info object.
 */
static void remove_item_from_jobs(slave_job_item *job_item,
                                  Slave_worker *worker, Relay_log_info *rli) {
  Log_event *ev = job_item->data;
  mysql_mutex_lock(&worker->jobs_lock);
  worker->jobs.de_queue(job_item);
  /* possible overfill */
  if (worker->jobs.get_length() == worker->jobs.capacity - 1 &&
      worker->jobs.overfill == true) {
    worker->jobs.overfill = false;
    // todo: worker->hungry_cnt++;
    mysql_cond_signal(&worker->jobs_cond);
  }
  mysql_mutex_unlock(&worker->jobs_lock);
  /* statistics */
  const auto jobs_length = worker->jobs.get_length();
  /* todo: convert to rwlock/atomic write */
  mysql_mutex_lock(&rli->pending_jobs_lock);
  rli->pending_jobs--;
  rli->mts_pending_jobs_size -= ev->common_header->data_written;
  assert(rli->mts_pending_jobs_size < rli->mts_pending_jobs_size_max);
  ...
```

另一个是在 **append_item_to_jobs** 函数中，如下面的代码片段所示：

```c++
/**
   Coordinator enqueues a job item into a Worker private queue.
   @param job_item  a pointer to struct carrying a reference to an event
   @param worker    a pointer to the assigned Worker struct
   @param rli       a pointer to Relay_log_info of Coordinator
   @return false Success.
           true  Thread killed or worker stopped while waiting for
                 successful enqueue.
*/
bool append_item_to_jobs(slave_job_item *job_item, Slave_worker *worker,
                         Relay_log_info *rli) {
  THD *thd = rli->info_thd;
  size_t ret = Slave_jobs_queue::error_result;
  size_t ev_size = job_item->data->common_header->data_written;
  ulonglong new_pend_size;
  PSI_stage_info old_stage;
  assert(thd == current_thd);
  mysql_mutex_lock(&rli->pending_jobs_lock);
  new_pend_size = rli->mts_pending_jobs_size + ev_size;
  bool big_event = (ev_size > rli->mts_pending_jobs_size_max);
  ...
```

*remove_item_from_jobs* 和 *append_item_to_jobs* 函数主要利用 *pending_jobs_lock* 锁存器来管理作业队列。*append_item_to_jobs* 函数由调度线程执行，每个事件调用一次以将其入队到 worker 队列。相比之下，*remove_item_from_jobs* 由 worker 线程执行，每次事件移除时调用一次。

在高吞吐量场景中，调度线程频繁调用 *append_item_to_jobs* 将事件入队，而众多 worker 线程同时调用 *remove_item_from_jobs* 将它们出队。这导致显著的锁存器争用，因为两个函数都涉及获取和释放锁存器。在事件处理速率达到每秒数十万的情况下，调度线程和 worker 线程之间的锁存器争用可能变得严重。

频繁获取和释放锁存器会导致上下文切换。在 NUMA 环境中，这些上下文切换导致 NUMA 节点之间的缓存迁移，导致回放效率降低。

在解决了与 *pending_jobs_lock* 相关的锁存器瓶颈并应用在线 MySQL 从库的配置参数后，重点转向评估在 BIOS 中禁用 NUMA 是否能提高 MySQL 从库回放性能。下图比较了不同 NUMA 配置下 MySQL 从库的平衡回放速度。

<img src="../media/image-20240829113628989.png" alt="image-20240829113628989" style="zoom:150%;" />

图 10-21. 通过解决 NUMA 不兼容瓶颈，在 BIOS 中禁用 NUMA 可提高 MySQL 从库回放速度。

如图所示，在解决 NUMA 不兼容瓶颈后，在 x86 架构上的 BIOS 中禁用 NUMA 可提高 MySQL 从库回放的速度。

### 10.3.6 "双一"对回放性能的影响

"双一"是指 binlog 和 redo log 的实时磁盘刷新，这是 MySQL 崩溃恢复的关键技术。测试表明，"双一"明显影响 MySQL 从库回放速度。下图显示，在相同条件下，禁用"双一"将平衡回放速度提高到 810,000 tpmC，而启用它则将回放速度降低到约 700,000 tpmC。因此，禁用"双一"使回放速度提高了 15.7%。

<img src="../media/image-20240829113649785.png" alt="image-20240829113649785" style="zoom:150%;" />

图 10-22. 关闭"双一"可实现更好的回放速度。

### 10.3.7 Binlog 行镜像格式对回放性能的影响

减小 binlog 的大小理论上有助于提高 MySQL 回放速度。下图给出使用 *binlog_row_image=minimal* 和 *binlog_row_image=full* 在平衡回放速度方面的比较：

<img src="../media/image-20240829113734608.png" alt="image-20240829113734608" style="zoom:150%;" />

图 10-23. 使用 binlog_row_image=minimal 实现更好的回放速度。

当对 binlog 使用 full 模式时，MySQL 实现的平衡回放速度刚刚超过 790,000 tpmC。然而，切换到 minimal 模式将此速度提高到超过 890,000 tpmC，提高了显著的 13%。这种改进突出表明，设置 *binlog_row_image=minimal*——显著减小了 binlog 大小——可以提高 MySQL 从库的回放速度。然而，需要注意的是，此设置在某些场景中也可能带来数据恢复不完整的风险。

### 10.3.8 Performance Schema 对回放性能的影响

由于从库回放涉及与大量内存的广泛交互，启用 Performance Schema 会明显影响 MySQL 从库的回放速度。这是启用 Performance Schema 后捕获的火焰图。

![](../media/22b839631e0b6d49bc9c5810488d9336.png)

图 10-24. my_malloc 调用 PFS 相关函数产生的大量开销。

如图所示，与 my_malloc 调用 PFS 相关函数相关的开销是巨大的，并且分布在多个区域，突出了一个重要的瓶颈。广泛的测试结果表明，与 MySQL 主库执行相比，启用 Performance Schema 对 MySQL 从库回放的影响要大得多。有关更详细的信息，请参阅第 11.5.5 节。

## 10.4 组复制中最大回放速度的探索

使用修改后的 tpcc-mysql，高吞吐量测试可以识别组复制从库何时落后于 MySQL 主库。下图给出详细的测试结果。测试设置包括：禁用"双一"，使用 jemalloc 4.5，在不同机器上部署 MySQL 从库和主库，将 binlog 行镜像格式设置为 minimal，并运行测试 300 秒。

<img src="../media/image-20240829113824280.png" alt="image-20240829113824280" style="zoom:150%;" />

图 10-25. 组复制中的最大回放速度。

该图说明了在并发级别为 100、150、200、300 和 400 时 MySQL 主库的吞吐量。在图中，浅灰色表示 MySQL 主库和从库的速度是同步的，而深灰色表示 MySQL 从库略微落后于 MySQL 主库。除了在 200 并发时观察到 2 秒延迟的情况外，MySQL 从库通常与主库保持同步。

需要注意，最大回放速度与并发直接相关。在 150 并发时，吞吐量很高，延迟几乎为零，而在 200 并发时，吞吐量略有降低，延迟为 2 秒。

通过更好的硬件，最大回放速度可能会有进一步的提升，但测试结果已经非常接近可实现的最大值。

[下一章](Part4.md)
