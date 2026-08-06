# 第8章：优化 MySQL 8.0：下一级改进

本章探讨了 MySQL 8.0 改进中发现的优化机会和解决方案，包括 SQL 层改进、InnoDB 存储引擎增强、事务限流机制以及低并发下读写性能的改进。

## 8.1 SQL 层的改进

解决了与查询执行计划相关的特定场景中的性能退化问题。同时改进了 binlog 组提交中激活用户线程的机制，进一步提高了效率。

### 8.1.1 解决查询执行计划中的性能退化

在 MySQL 8.0.27 上进行二次开发时，使用 BenchmarkSQL 进行的 TPC-C 测试变得不稳定。吞吐量迅速下降，优化过程复杂化。对官方版本的信任导致最初忽视了这个问题，尽管测试困难重重。

直到用户报告升级后出现显著性能下降，我们才开始认真对待这个问题。来自用户的可靠反馈表明，虽然 MySQL 8.0.25 表现良好，但升级到 MySQL 8.0.29 后性能大幅下降。这一关键信息表明存在性能问题。

同时，确认了 MySQL 8.0.27 中的性能退化问题与 MySQL 8.0.29 中的问题相同。MySQL 8.0.27 专门针对 trx-sys 进行了两次可扩展性优化，理论上应该提高了吞吐量。回顾 trx-sys 中 latch 分片对性能的影响：

<img src="../media/image-20240829102323261.png" alt="image-20240829102323261" style="zoom:150%;" />

图 8-1. 不同并发级别下 trx-sys 中 latch 分片的影响。

让我们继续研究 trx-sys latch 分片优化与 MySQL 8.0.27 发布版本之间的吞吐量和并发性对比。具体细节如下图所示：

<img src="../media/image-20240829102344815.png" alt="image-20240829102344815" style="zoom:150%;" />

图 8-2. MySQL 8.0.27 发布版本中的性能退化。

从图中可以明显看出，MySQL 8.0.27 发布版本在低并发条件下的性能退化很严重，峰值性能明显下降。这与用户关于吞吐量下降的反馈一致，并且使用 BenchmarkSQL 很容易复现。

MySQL 8.0.27 发布版本已经存在这个问题，而较早的 MySQL 8.0.25 发布版本却没有。利用这些信息，目标是识别导致性能退化的特定 git commit。找到导致性能退化的 git commit 是一个复杂的过程，通常涉及二分查找。经过大量测试，最初将范围缩小到一个特定的 commit。然而，这个 commit 包含数万行代码，几乎不可能精确定位导致问题的确切代码段。后来发现这个 commit 是从特定分支的集体合并。这允许进一步细分，最终在以下内容中识别出问题的根本原因：

```c++
commit 9a13c1c6971f4bd56d143179ecfb34cca8ecc018
Author: Steinar H. Gunderson <steinar.gunderson@oracle.com>
Date:   Tue Jun 8 15:14:35 2021 +0200

    Bug #32976857: REMOVE QEP_TAB_STANDALONE [range optimizer, noclose]

    Remove the QEP_TAB dependency from test_quick_select() (ie., the range
    optimizer).

    Change-Id: Ie0fcce71dfc813920711c43c3d62635dae0d7d20
```

使用 commit 信息，编译了两个版本，并识别了在 TPC-C 测试中执行异常缓慢的 SQL 查询。使用 *'explain'* 分析了这些慢 SQL 查询的执行计划。具体细节如下图所示：

![](../media/47c39411c3240713c75e848ef5ef4b59.png)

图 8-3. *'explain'* 中 rows 指示的异常。

从图中可以看出，大多数执行计划是相同的，除了 *'rows'* 列。在正常版本中，*'rows'* 列显示刚刚超过 200，而在有问题的版本中，显示超过 1,000,000。在不断简化 SQL 后，最终识别了一个高度代表性的 SQL 查询。具体细节如下图所示：

![](../media/19a92bf7065627b28a3403c000eba095.png)

图 8-4. SQL 执行结果与 *'explain'* 输出之间的显著差异。

根据从 '*explain*' 获得的 *Filter* 信息，构建了图中显示的最后一个查询。图中显示，虽然最后一个查询只返回了 193 行，但 '*explain*' 显示 *'rows'* 超过 117 万行。这种差异突显了一个复杂的问题，因为并非所有 MySQL 开发人员都完全理解执行计划。幸运的是，识别导致性能退化的 commit 为解决问题提供了关键基础。尽管有了这些信息，解决问题相对简单，但从 SQL 语句本身分析根本原因证明要困难得多。

让我们继续深入分析这个问题。下图显示了特定 SQL 查询的 '*explain*' 结果：

![](../media/63cf3b8bd556c6b1ce5a0436883c8f7b.png)

图 8-5. 代表问题的示例 SQL 查询。

从图中可以看出，行数仍然很大，表明这个 SQL 查询具有代表性。

编译了两个不同的 debug 版本的 MySQL：一个有异常，一个正常。使用 debug 版本通过 debug 跟踪捕获有用的函数调用关系。在异常版本上执行有问题的 SQL 语句时，相关的 debug 跟踪信息如下：

![](../media/149ca62fd014bd12b60c77573a49757d.gif)

图 8-6. 异常版本的 debug 跟踪信息。

类似地，对于正常版本，相关的 debug 跟踪信息如下：

![](../media/cba8723ae722e8d2d13b94e0cf1fda7a.png)

图 8-7. 正常版本的 debug 跟踪信息。

比较上面两个图，可以注意到正常版本在绿色框内包含额外的内容，表明在正常版本中应用了条件，而异常版本缺少这些条件。要理解为什么异常版本缺少这些条件，需要在 *get_full_func_mm_tree* 函数中添加额外的跟踪信息以捕获造成这种差异的具体细节。

添加额外的跟踪信息后，异常版本的 debug 跟踪结果如下：

![](../media/b89dd5988d7d7ead6923dbc2d20e146c.png)

图 8-8. 异常版本的补充 debug 跟踪信息。

正常版本的 debug 跟踪结果如下：

![](../media/5136495bbdfbe2cefac98d74bd36a88f.png)

图 8-9. 正常版本的补充 debug 跟踪信息。

比较上面两个图，观察到显著差异。在正常版本中，*param_comp* 的值是 16140901064495857660，而在异常版本中是 16140901064495857661，相差 1。要理解这种差异，让我们首先检查 *param_comp* 值是如何计算的，如下面的代码片段所详细说明：

```c++
static SEL_TREE *get_full_func_mm_tree(THD *thd, RANGE_OPT_PARAM *param,
                                       table_map prev_tables,
                                       table_map read_tables,
                                       table_map current_table,
                                       bool remove_jump_scans, Item *predicand,
                                       Item_func *op, Item *value, bool inv) {
  SEL_TREE *tree = nullptr;
  SEL_TREE *ftree = nullptr;
  const table_map param_comp = ~(prev_tables | read_tables | current_table);
  DBUG_TRACE;
  ...
```

从代码可以清楚地看出，*param_comp* 是使用三个变量的按位 OR 运算计算的，然后是按位 NOT 运算。差值为 1 表明至少有一个变量不同，有助于缩小问题范围。

计算涉及三个具有很长值的 *table_map* 变量，使得普通计算器不足以完成任务，并且过程过于复杂，无法在此详细说明。

关键是 debug 跟踪揭示了关键差异。再加上通过识别导致性能差异的 Git commit 提供的信息，分析根本原因不再困难。

这是最终的修复补丁，详细如下：

![](../media/da7d6c8c12d18b915018939970d2b911.png)

图 8-10. 解决查询执行计划性能退化的最终补丁。

在调用 *test_quick_select* 函数时，重新引入 *const_table* 和 *read_tables* 变量（与前面讨论的变量相关）。这确保执行计划中的过滤条件不会被忽略。

将上述补丁应用于 MySQL 8.0.27 后，性能退化问题得到解决。在应用补丁之前和之后，在各种并发级别下进行了比较 TPC-C 吞吐量的测试。具体细节如下图所示：

<img src="../media/image-20240829102642856.png" alt="image-20240829102642856" style="zoom:150%;" />

图 8-11. 补丁对解决性能退化的效果。

从图中可以明显看出，应用补丁后，在低并发条件下吞吐量和峰值性能显著提高。然而，在高并发条件下，吞吐量不仅没有增加，反而下降了，这可能是由于 MVCC ReadView 的可扩展性瓶颈。

解决 MVCC ReadView 可扩展性问题后，重新评估此补丁的影响，如下图所示：

<img src="../media/image-20240829102703396.png" alt="image-20240829102703396" style="zoom:150%;" />

图 8-12. 解决 MVCC ReadView 可扩展性问题后补丁的实际效果。

从图中可以明显看出，这个补丁显著提高了 MySQL 的吞吐量。这个案例表明，可扩展性问题可能会破坏某些优化。要科学评估优化的有效性，必须事先解决大多数可扩展性问题，以实现更准确的评估。

最后，让我们检查 TPC-C 长期稳定性测试的结果。下图显示了在 100 并发下进行的 8 小时测试的结果，在不同小时（其中 1 ≤ n ≤ 8）捕获吞吐量。

<img src="../media/image-degrade4.png" alt="image-degrade4" style="zoom:150%;" />

图 8-13. 稳定性测试对比：MySQL 8.0.27 vs. 改进的 MySQL 8.0.27。

从图中可以明显看出，应用补丁后，吞吐量下降的速度已被显著缓解。MySQL 8.0.27 版本经历了急剧的吞吐量下降，未能满足 TPC-C 测试的稳定性要求。然而，应用补丁后，MySQL 的性能恢复正常。

直接解决这个问题面临相当大的挑战，特别是对于不熟悉查询执行计划的 MySQL 开发人员。使用逻辑推理和系统方法来识别和解决问题出现前后的代码差异是一种更优雅的解决问题的方法，尽管它很复杂。

值得注意的是，在应用补丁后没有遇到回归测试问题，表现出高稳定性，并为未来的性能改进提供了坚实的基础。目前，MySQL 8.0.40 仍然没有解决这个问题，表明 MySQL 的测试系统可能存在不足。鉴于 MySQL 数据库的复杂性，用户在升级时应谨慎行事，并考虑使用 TCPCopy [65] 等工具来避免潜在的回归测试问题。

### 8.1.2 改进 Binlog 组提交的可扩展性

binlog 组提交机制相当复杂，这种复杂性使得识别其固有的性能问题变得具有挑战性。

首先，使用 *perf* 工具在 500 并发的 TPC-C 测试期间捕获性能问题，如下图所示：

![](../media/66a031ab14d56289d0987c65c73323af.png)

图 8-14. *_pthread_mutex_cond_lock* 瓶颈揭示性能问题。

显然 *_pthread_mutex_cond_lock* 是一个重要的瓶颈，约占 9.5% 的开销。虽然 *perf* 没有直接指出确切的问题，但它表明存在这个瓶颈。

为了解决这个问题，对 MySQL 内部进行了深入探索，以揭示导致此性能瓶颈的因素。使用常规的二分查找方法和最小日志记录来识别执行期间产生显著开销的函数或代码段。选择最小日志记录方法是为了减少诊断问题根本原因时的性能干扰。过多的日志记录会破坏性能分析，虽然有些人可能使用 MySQL 的内部机制进行故障排除，但这些机制本身通常会引入大量的性能开销。

经过彻底调查，在以下代码段中识别出瓶颈。

```c++
  /*
    If the queue was not empty, we're a follower and wait for the
    leader to process the queue. If we were holding a mutex, we have
    to release it before going to sleep.
  */
  if (!leader) {
    CONDITIONAL_SYNC_POINT_FOR_TIMESTAMP("before_follower_wait");
    mysql_mutex_lock(&m_lock_done);
    ...
    ulonglong start_wait_time = my_micro_time();
    while (thd->tx_commit_pending) {
      if (stage == COMMIT_ORDER_FLUSH_STAGE) {
        mysql_cond_wait(&m_stage_cond_commit_order, &m_lock_done);
      } else {
        mysql_cond_wait(&m_stage_cond_binlog, &m_lock_done);
      }
    }
    ulonglong end_wait_time = my_micro_time();
    ulonglong wait_time = end_wait_time - start_wait_time;
    if (wait_time > 100000) {
        fprintf(stderr, "wait too long:%llu\n", wait_time);
    }
    mysql_mutex_unlock(&m_lock_done);
    return false;
  }
```

大量出现的 'wait too long' 输出表明瓶颈已经暴露。为了调查为什么报告 'wait too long'，添加并相应修改了日志。参见下面的具体代码：

```c++
 /*
    If the queue was not empty, we're a follower and wait for the
    leader to process the queue. If we were holding a mutex, we have
    to release it before going to sleep.
  */
  if (!leader) {
    CONDITIONAL_SYNC_POINT_FOR_TIMESTAMP("before_follower_wait");
    mysql_mutex_lock(&m_lock_done);
    ...
    ulonglong start_wait_time = my_micro_time();
    while (thd->tx_commit_pending) {
      if (stage == COMMIT_ORDER_FLUSH_STAGE) {
        mysql_cond_wait(&m_stage_cond_commit_order, &m_lock_done);
      } else {
        mysql_cond_wait(&m_stage_cond_binlog, &m_lock_done);
      }
      fprintf(stderr, "wake up thread:%p,total wait time:%llu, stage:%d\n",
              thd, my_micro_time() - start_wait_time, stage);
    }
    ulonglong end_wait_time = my_micro_time();
    ulonglong wait_time = end_wait_time - start_wait_time;
    if (wait_time > 100000) {
        fprintf(stderr, "wait too long:%llu for thread:%p\n", wait_time, thd);
    }
    mysql_mutex_unlock(&m_lock_done);
    return false;
  }
```

经过另一轮测试后，观察到一个奇特的现象：当出现 'wait too long' 消息时，'wake up thread' 日志显示许多用户线程被多次唤醒。

问题被追溯到 *thd->tx_commit_pending* 值没有改变，导致线程反复重新进入等待过程。进一步检查显示此变量变为 false 的条件，如以下代码所示：

```c++
void Commit_stage_manager::signal_done(THD *queue, StageID stage) {
  mysql_mutex_lock(&m_lock_done);
  for (THD *thd = queue; thd; thd = thd->next_to_commit) {
    thd->tx_commit_pending = false;
    thd->rpl_thd_ctx.binlog_group_commit_ctx().reset();
  }
  /* if thread belong to commit order wake only commit order queue threads */
  if (stage == COMMIT_ORDER_FLUSH_STAGE)
    mysql_cond_broadcast(&m_stage_cond_commit_order);
  else
    mysql_cond_broadcast(&m_stage_cond_binlog);
  mysql_mutex_unlock(&m_lock_done);
}
```

从代码可以明显看出，*thd->tx_commit_pending* 在 *signal_done* 函数中被设置为 false。然后 *mysql_cond_broadcast* 函数激活所有等待的线程，导致类似于惊群问题的情况。当所有先前等待的用户线程被激活时，它们检查 tx_commit_pending 是否已设置为 false。如果已经设置，它们继续处理；否则，它们继续等待。

尽管 binlog 组提交机制很复杂，但直接的分析可以识别根本原因：不应该被激活的线程被触发，导致每次激活都会产生不必要的上下文切换。

在一次测试中，收集了有关用户线程进入等待状态次数的额外统计信息。详细信息如下图所示：

<img src="../media/image-20240829103131857.png" alt="image-20240829103131857" style="zoom:150%;" />

图 8-15. 被激活 1、2、3 次的线程统计。

等待一次是正常的，表示 100% 的效率。等待两次表示 50% 的效率，等待三次表示 33.3% 的效率。根据图中的数据，计算出的总体激活效率为 52.7%。

为了解决这个问题，理想的解决方案是具有 100% 效率的多播激活机制，其中 tx_commit_pending 设置为 false 的用户线程被一起激活。然而，实现这需要深入理解 binlog 组提交背后的复杂逻辑。

在这种情况下，使用点对点激活机制，实现 100% 的效率，但引入了显著的系统调用开销。下图说明了优化前后 TPC-C 吞吐量与并发性之间的关系。

<img src="../media/image-20240829103236734.png" alt="image-20240829103236734" style="zoom:150%;" />

图 8-16. innodb_thread_concurrency=128 时组提交优化的影响。

从图中可以明显看出，当 innodb_thread_concurrency=128 时，binlog 组提交的优化在高并发下显著提高了吞吐量。

重要的是要注意，此优化的有效性可能因配置设置和特定场景等因素而异。然而，总体而言，它显著提高了吞吐量，尤其是在高并发条件下。

下面是使用标准配置优化前后 TPC-C 吞吐量和并发性的对比：

<img src="../media/image-20240829103259992.png" alt="image-20240829103259992" style="zoom:150%;" />

图 8-17. 使用标准配置的组提交优化影响。

从图中可以清楚地看出，与前一个相比，此优化不太明显，但它仍然显示出整体改进。广泛的测试表明，MySQL 的可扩展性越差，binlog 组提交优化的有效性就越显著。

同时，先前识别的 *_pthread_mutex_cond_lock* 瓶颈在优化后得到了显著缓解，如下图所示：

![](../media/8f9080ee71094d948ab7592b449954bb.png)

图 8-18. *_pthread_mutex_cond_lock* 瓶颈的缓解。

总之，此优化有助于解决与 binlog 组提交相关的可扩展性问题。

## 8.2 增强 InnoDB 存储引擎

### 8.2.1 MVCC ReadView：已识别的问题

任何 MVCC 方案的关键组成部分是快速确定哪些元组对哪些事务可见的机制。通过构建一个 ReadView (RV) 向量来创建事务的快照，该向量保存所有小于事务 TXID 的并发事务的 TXID。获取快照的成本随并发事务的数量线性增加，即使事务仅读取由单个已提交事务写入的元组，这也突显了已知的可扩展性限制 [7]。

在理解 MVCC ReadView 机制的可扩展性问题后，让我们检查 MySQL 如何实现 MVCC ReadView。在 Read Committed 隔离级别下，在读取数据的过程中，InnoDB 存储引擎触发 ReadView 的获取。ReadView 数据结构的部分截图如下所示：

```c++
private:
  // Disable copying
  ReadView(const ReadView &);
  ReadView &operator=(const ReadView &);
private:
  /** The read should not see any transaction with trx id >= this
  value. In other words, this is the "high water mark". */
  trx_id_t m_low_limit_id;
  /** The read should see all trx ids which are strictly
  smaller (<) than this value.  In other words, this is the
  low water mark". */
  trx_id_t m_up_limit_id;
  /** trx id of creating transaction, set to TRX_ID_MAX for free
  views. */
  trx_id_t m_creator_trx_id;
  /** Set of RW transactions that was active when this snapshot
  was taken */
  ids_t m_ids;
  /** The view does not need to see the undo logs for transactions
  whose transaction number is strictly smaller (<) than this value:
  they can be removed in purge if not needed by other views */
  trx_id_t m_low_limit_no;
  ...
```

这里，*m_ids* 是类型为 *ids_t* 的数据结构，与 *std::vector* 非常相似。具体说明如下：

```c++
  /** This is similar to a std::vector but it is not a drop
  in replacement. It is specific to ReadView. */
  class ids_t {
    typedef trx_ids_t::value_type;
    /**
    Constructor */
    ids_t() : m_ptr(), m_size(), m_reserved() {}
    /**
    Destructor */
    ~ids_t() { ut::delete_arr(m_ptr); }
    /** Try and increase the size of the array. Old elements are copied across.
    It is a no-op if n is < current size.
    @param n            Make space for n elements */
    void reserve(ulint n);
    ...
```

MVCC ReadView 可见性判断算法，具体参考下面的 *changes_visible* 函数：

```c++
  /** Check whether the changes by id are visible.
  @param[in]    id      transaction id to check against the view
  @param[in]    name    table name
  @return whether the view sees the modifications of id. */
  [[nodiscard]] bool changes_visible(trx_id_t id,
                                     const table_name_t &name) const {
    ut_ad(id > 0);
    if (id < m_up_limit_id || id == m_creator_trx_id) {
      return (true);
    }
    check_trx_id_sanity(id, name);
    if (id >= m_low_limit_id) {
      return (false);
    } else if (m_ids.empty()) {
      return (true);
    }
    const ids_t::value_type *p = m_ids.data();
    return (!std::binary_search(p, p + m_ids.size(), id));
 }
```

从代码可以看出，当并发性较低时，可见性算法工作得很高效。然而，随着并发性的增加，使用二分查找来确定可见性的效率显著降低，尤其是在 NUMA 环境中。

### 8.2.2 增强 MVCC ReadView 可扩展性的解决方案

这里有两种改进可扩展性的基本方法 [58]：

*首先，找到一种改进复杂度的算法，使得每个额外的连接不会线性增加快照计算成本。*

*其次，为每个连接执行更少的工作，希望减少总耗时，以至于即使在高连接数时，总时间仍然足够小，影响不大（即减少常数因子）。*

对于第一种解决方案，采用基于 Commit Sequence Numbers (CSN) 的多版本可见性算法提供了好处 [7]：*通过将快照转换为 CSN 而不是维护事务 ID 列表，可以减少获取快照的成本。* 具体来说，在 Read Committed 隔离级别下，不需要为每次读取操作复制活跃事务列表，从而提高可扩展性。

考虑到实现的复杂性，本书选择第二种解决方案，它直接修改 MVCC ReadView 数据结构以缓解 MVCC ReadView 可扩展性问题。

### 8.2.3 改进 MVCC ReadView 数据结构

在 ReadView 结构中，原始方法使用 vector 来存储活跃事务列表。现在，已更改为以下数据结构：

```c++
class ReadView {
 ...
 private:
  // Disable copying
  ReadView &operator=(const ReadView &);
 public:
  bool skip_view_list{false};
 private:
  unsigned char top_active[MAX_TOP_ACTIVE_BYTES];
  trx_id_t m_short_min_id;
  trx_id_t m_short_max_id;
  bool m_has_short_actives;
  /** The read should not see any transaction with trx id >= this
  value. In other words, this is the "high water mark". */
  trx_id_t m_low_limit_id;
  /** The read should see all trx ids which are strictly
  smaller (<) than this value.  In other words, this is the low water mark". */
  trx_id_t m_up_limit_id;
  /** trx id of creating transaction, set to TRX_ID_MAX for free views. */
  trx_id_t m_creator_trx_id;
  /** Set of RW transactions that was active when this snapshot
  was taken */
  ids_t m_long_ids;
  ...
```

此外，在相关接口函数中进行了相应的代码修改，因为数据结构的更改需要对这些函数内部的代码进行调整。

这个新的 MVCC ReadView 数据结构可以看作是一个混合数据结构，如下图所示。

![](../media/a54faa33502b8c17066b1e2af09bdbb0.png)

图 8-19. 适用于 MVCC ReadView 中活跃事务列表的新混合数据结构。

有关更详细的解释，请参阅第 4.2.8 章关于混合数据结构的内容。

通常，在线事务是短事务而不是长事务，并且事务 ID 持续增加。为了利用这些特性，使用混合数据结构：静态数组用于连续的短事务 ID，向量用于长事务。使用 2048 字节的数组，最多可以存储 16,384 个连续的活跃事务 ID，每个位代表一个事务 ID。

最小短事务 ID 用于区分短事务和长事务。小于此最小值的 ID 进入长事务向量，而等于或大于它的 ID 放置在短事务数组中。对于 changes_visible 中的 ID，如果它低于最小短事务 ID，则直接查询向量，由于长事务的数量通常很小，因此效率很高。如果 ID 等于或高于最小短事务 ID，则执行按位查询，时间复杂度为 O(1)，而之前的复杂度为 O(log n)。这种改进提高了效率并减少了 NUMA 节点之间的缓存迁移，因为 O(1) 查询通常在单个 CPU 时间片内完成。

除了前面提到的转换之外，类似的修改也应用于全局事务活跃列表。用于此列表的原始数据结构如以下代码片段所示：

```c++
  /** Array of Read write transaction IDs for MVCC snapshot. A ReadView would
  take a snapshot of these transactions whose changes are not visible to it.
  We should remove transactions from the list before committing in memory and
  releasing locks to ensure right order of removal and consistent snapshot. */
  trx_ids_t rw_trx_ids;
```

现在已更改为以下数据结构：

```c++
  /** Array of Read write transaction IDs for MVCC snapshot. A ReadView would
  take a snapshot of these transactions whose changes are not visible to it.
  We should remove transactions from the list before committing in memory and
  releasing locks to ensure right order of removal and consistent snapshot. */
  trx_ids_t long_rw_trx_ids;
  unsigned char short_rw_trx_ids_bitmap[MAX_SHORT_ACTIVE_BYTES];
  int short_rw_trx_valid_number;
  trx_id_t min_short_valid_id;
  trx_id_t max_short_valid_id
```

在 *short_rw_trx_ids_bitmap* 结构中，*MAX_SHORT_ACTIVE_BYTES* 设置为 65536，理论上最多可容纳 524,288 个连续的短事务 ID。如果超过限制，最旧的短事务 ID 将转换为长事务并存储在 *long_rw_trx_ids* 中。全局长短事务由 *min_short_valid_id* 区分：小于此值的 ID 被视为全局长事务，而等于或大于的 ID 被视为全局短事务。

在从全局活跃事务列表复制的过程中，*short_rw_trx_ids_bitmap* 结构每个事务 ID 仅使用一位，与原生 MySQL 解决方案相比，复制效率要高得多。例如，对于 1000 个活跃事务，原生 MySQL 版本需要复制至少 8000 字节，而优化的解决方案可能只需要几百字节。这导致复制效率显著提高。

实施这些修改后，进行了性能对比测试以评估 MVCC ReadView 优化的有效性。下图显示了修改 MVCC ReadView 数据结构前后在不同并发级别下的 TPC-C 吞吐量对比。

<img src="../media/image-20240829104222478.png" alt="image-20240829104222478" style="zoom:150%;" />

图 8-20. 在 NUMA 中采用新混合数据结构前后的性能对比。

从图中可以明显看出，这种转换主要优化了可扩展性，并提高了 MySQL 在 NUMA 环境中的峰值吞吐量。可以使用 *perf* 等工具进一步分析优化前后的性能对比。下面是优化前在 300 并发下来自 *perf* 的截图：

![](../media/bb111919295b7678530a1adcfa8b7d29.png)

图 8-21. *perf* 截图中观察到的与 Latch 相关的瓶颈。

从图中可以看出，前两个瓶颈很严重，约占 33% 的开销。优化后，300 并发下的 *perf* 截图如下：

![](../media/47f963cdd950abd91a459ffb66a3744e.png)

图 8-22. 与 Latch 相关的瓶颈显著缓解。

优化后，如上面的截图所示，之前排名前两位的瓶颈的比例已显著降低。

为什么改变 MVCC ReadView 数据结构能显著增强可扩展性？这是因为访问这些结构涉及获取全局 latch。优化数据结构加速了对关键资源的访问，减少了并发冲突并最小化了 NUMA 节点间的缓存迁移。

原生 MVCC ReadView 使用 vector 来存储活跃事务列表。在高并发场景中，这个列表可能变得很大，导致工作集更大。在 NUMA 环境中，查询和复制都可能变慢，可能导致单个 CPU 时间片错过其截止时间，并导致显著的上下文切换成本。这方面的理论基础如下 [21]：

*在逻辑操作中间发生的上下文切换从缓存中驱逐可能更大的工作集。当暂停的线程恢复执行时，它会浪费时间恢复被驱逐的工作集。*

接下来评估 ARM 架构下的吞吐量改进。详细信息如下图所示：

<img src="../media/image-20240829104512068.png" alt="image-20240829104512068" style="zoom:150%;" />

图 8-23. ARM 架构下的吞吐量改进。

从图中可以明显看出，在 ARM 架构下也有显著的改进。广泛的测试数据证实，无论架构是 ARM 还是 x86，MVCC ReadView 优化在 NUMA 环境中都能产生明显的好处。

这种优化在 SMP 环境中能带来多大的改进？

<img src="../media/image-20240829104533718.png" alt="image-20240829104533718" style="zoom:150%;" />

图 8-24. 在 SMP 中采用新混合数据结构前后的性能对比。

从图中可以观察到，在绑定到 NUMA 节点 0 后，MVCC ReadView 优化的改进并不显著。这表明该优化主要增强了 NUMA 架构中的可扩展性。

在实际的 MySQL 使用中，防止过多的用户线程进入 InnoDB 存储引擎可以显著减小全局活跃事务列表的大小。这种事务限流机制有效地补充了 MVCC ReadView 优化，提高了整体性能。结合下一节讨论的双重 latch 避免，下图中的 TPC-C 测试结果清楚地展示了这些改进。

<img src="../media/image-20240829104554155.png" alt="image-20240829104554155" style="zoom:150%;" />

图 8-25. 使用事务限流机制的 BenchmarkSQL 中的最大 TPC-C 吞吐量。

### 8.2.4 避免双重 Latch 问题

在 MVCC ReadView 优化后的测试中，在极高并发条件下观察到吞吐量的明显下降。具体细节如下图所示：

<img src="../media/image-20240829104639402.png" alt="image-20240829104639402" style="zoom:150%;" />

图 8-26. 并发级别超过 500 时的性能退化。

从图中可以看出，一旦并发超过 500，吞吐量就会显著下降。问题被追溯到频繁获取 *trx-sys* latch，如下面的代码片段所示：

```c++
    } else if (trx->isolation_level <= TRX_ISO_READ_COMMITTED &&
               MVCC::is_view_active(trx->read_view)) {
      mutex_enter(&trx_sys->mutex);
      trx_sys->mvcc->view_close(trx->read_view, true);
      mutex_exit(&trx_sys->mutex);
    }
```

另一个代码片段如下所示：

```c++
  if (lock_type != TL_IGNORE && trx->n_mysql_tables_in_use == 0) {
    trx->isolation_level =
        innobase_trx_map_isolation_level(thd_get_trx_isolation(thd));
    if (trx->isolation_level <= TRX_ISO_READ_COMMITTED &&
        MVCC::is_view_active(trx->read_view)) {
      /* At low transaction isolation levels we let
      each consistent read set its own snapshot */
      mutex_enter(&trx_sys->mutex);
      trx_sys->mvcc->view_close(trx->read_view, true);
      mutex_exit(&trx_sys->mutex);
    }
  }
```

InnoDB 在视图关闭过程中引入了全局 trx-sys latch，影响了高并发下的可扩展性。为了解决这个问题，尝试移除全局 latch。其中一个修改如下面的代码片段所示：

```c++
   } else if (trx->isolation_level <= TRX_ISO_READ_COMMITTED &&
               MVCC::is_view_active(trx->read_view)) {
      trx_sys->mvcc->view_close(trx->read_view, false);
}
```

另一个修改如下面的代码片段所示：

```c++
  if (lock_type != TL_IGNORE && trx->n_mysql_tables_in_use == 0) {
    trx->isolation_level =
        innobase_trx_map_isolation_level(thd_get_trx_isolation(thd));
    if (trx->isolation_level <= TRX_ISO_READ_COMMITTED &&
        MVCC::is_view_active(trx->read_view)) {
      /* At low transaction isolation levels we let
      each consistent read set its own snapshot */
      trx_sys->mvcc->view_close(trx->read_view, false);
    }
  }
```

使用 MVCC ReadView 优化版本，比较修改前后的 TPC-C 吞吐量。详细信息如下图所示：

<img src="../media/image-20240829104851205.png" alt="image-20240829104851205" style="zoom:150%;" />

图 8-27. 消除双重 latch 瓶颈后的性能改进。

从图中可以明显看出，修改显著改善了高并发条件下的可扩展性。为了理解这种改进的原因，让我们使用 *perf* 工具进行进一步调查。下面是修改前在 2000 并发下的 *perf* 截图：

![](../media/ccb771014f600402fee72ca7134aea10.gif)

图 8-28. *perf* 截图中观察到的与 Latch 相关的瓶颈。

从图中可以明显看出，与 latch 相关的瓶颈相当明显。代码修改后，这是 3000 并发下的 *perf* 截图：

![](../media/ce8a6d4b9cd6df4c48cf914fae8a70d2.png)

图 8-29. 与 Latch 相关的瓶颈显著缓解。

即使在更高的并发级别（如 3000）下，瓶颈也不明显。这表明优化已有效缓解了与 latch 相关的性能问题，改善了极端条件下的可扩展性。

排除 *view_close* 函数调用前后的全局 latch 可以改善可扩展性，而包含它则在高并发下严重降低可扩展性。尽管 *view_close* 函数在其临界区内高效运行，但频繁获取在整个 *trx-sys* 子系统中使用的全局 *trx-sys* latch 会导致显著的争用和队头阻塞。这个问题被称为"双重 latch"问题，源于 *view_open* 和 *view_close* 都需要全局 *trx-sys* latch。值得注意的是，从最后阶段移除 latch 或使用新的 latch 可以显著缓解这个问题。

### 8.2.5 解释超线性性能现象

第 2.1 节描述了在 x86 NUMA 环境中 SysBench 读写测试期间观察到的吞吐量超线性扩展。在对 InnoDB 存储引擎进行改进后，当前调查检查这种超线性扩展效应是否仍然存在。使用改进版本的 MySQL 在同一环境中进行了测试。MySQL 实例 1 和 MySQL 实例 2 的单独 SysBench 测试结果如下图所示：

<img src="../media/image-20240829104924497.png" alt="image-20240829104924497" style="zoom:150%;" />

图 8-30. MVCC 优化后分别运行 MySQL 的吞吐量。

每个实例的吞吐量都有显著提高，MySQL 实例 1 实现了 524,381 QPS，MySQL 实例 2 达到了 553,008 QPS。合计起来，它们提供了 1,077,389 QPS，大大超过了之前的 328,168 QPS。

SysBench 用于同时评估这两个实例的读写性能，如下图所示。

<img src="../media/image-20240829104945257.png" alt="image-20240829104945257" style="zoom:150%;" />

图 8-31. MVCC 优化后一起运行 MySQL 的吞吐量。

从图中可以看出，一个实例实现了 289,702 QPS 的吞吐量，而另一个达到了 285,026 QPS。两个 MySQL 实例的合计吞吐量总计 574,728 QPS，与 MySQL 发布版本观察到的 546,429 QPS 非常接近。这说明了 Linux 操作系统在 NUMA 环境中有效调度多个进程的作用。

从数据中可以明显看出，两个 MySQL 实例同时运行的总吞吐量明显低于每个实例单独运行的吞吐量。有关详细的统计对比，请参考下图：

<img src="../media/image-20240829105004347.png" alt="image-20240829105004347" style="zoom:150%;" />

图 8-32. MVCC 优化后分别运行与一起运行的总吞吐量。

在对 InnoDB 存储引擎进行增强后，超线性扩展问题已得到解决，揭示了 MySQL 发布版本中的根本原因是 InnoDB 内设计不佳的 MVCC 机制。这个设计缺陷在 NUMA 环境中放大了问题，导致观察到的超线性扩展效应。

## 8.3 事务限流机制

根据论文"Staring into the Abyss: An Evaluation of Concurrency Control with One Thousand Cores" [22]，包括 MySQL 在内的中心化数据库难以充分利用数百个 CPU 核心。

为了解决可扩展性问题，传统方法使用线程池来限制数据库利用的 CPU 核心数量。本书介绍了一种事务限流机制，限制访问事务系统的线程数量，提供了一种替代方法来缓解可扩展性挑战。

### 8.3.1 Percona 线程池

一般来说，传统 MySQL 中的线程池主要有两个目的：

1. **缓解短连接风暴**：通过管理和重用线程，线程池有助于防止短期连接突然激增期间的系统过载。
2. **增强可扩展性**：线程池改善了可扩展性，特别是在高争用场景中，使 MySQL 能够更有效地利用可用的 CPU 核心。

以 Percona 的线程池为例，让我们检查线程池在改善 MySQL 可扩展性方面的成本效益。下图比较了使用改进版本的 MySQL 实施线程池前后的吞吐量和并发性。

<img src="../media/image-20240829105130499.png" alt="image-20240829105130499" style="zoom:150%;" />

图 8-33. 启用 Percona 线程池导致吞吐量明显下降。

从图中可以明显看出，采用线程池后吞吐量下降。这种下降归因于 Percona 线程池的高固有成本。此外，由于改进版本的 MySQL 已经实现了显著的可扩展性改进，Percona 线程池在改善 TPC-C 应用程序可扩展性方面的额外好处有限。

对于可扩展性较差的 MySQL 版本，线程池在解决可扩展性问题方面仍然有价值。如第 1 章所示，线程池的使用显著缓解了 MySQL 5.7 中的可扩展性问题。

基于广泛的测试，在解决了 MySQL 的大部分可扩展性问题后，观察到虽然 Percona 线程池在管理短期连接和高争用场景中仍然有效，但其整体有效性降低，甚至可能在其他情况下阻碍性能。

### 8.3.2 事务限流机制

中心化数据库由于其事务系统的限制而难以充分利用数百个 CPU 核心。为了解决这个问题，事务限流机制变得越来越重要。

MySQL 在其线程池中引入了"最大事务限制"功能以缓解性能退化 [31]。此功能限制并发事务的数量，通过减少重负载系统上的数据锁和死锁来提高吞吐量 [64]。这种方法可以启发类似的机制，在高并发场景中增加吞吐量，而不仅仅依赖传统的线程池。

对于 MySQL，事务限流的具体流程图如下：

![](../media/480170fff071f9764917821bb3a6ff6c.png)

图 8-34. MySQL 中的新事务限流机制。

在启动新事务之前，检查并发事务数量是否超过限制。如果超过，事务进入等待状态。否则，它进入事务系统。事务完成执行后，等待列表中的用户线程被激活以继续执行另一个事务。

实施这种事务限流机制后，验证 MySQL 的可扩展性。下图说明了使用限流机制与使用 Percona 线程池相比时 TPC-C 吞吐量与并发性之间的关系。

<img src="../media/image-20240829105242300.png" alt="image-20240829105242300" style="zoom:150%;" />

图 8-35. 事务限流机制的影响。

从图中可以看出，限流方法优于 Percona 线程池方法，并且这种优势是全面的。

下图描述了实施事务限流后进行的 TPC-C 可扩展性压力测试。测试在禁用 NUMA BIOS 的场景中进行，限制最多 512 个用户线程进入事务系统。

<img src="../media/image-20240829105258689.png" alt="image-20240829105258689" style="zoom:150%;" />

图 8-36. 使用事务限流机制的 BenchmarkSQL 中的最大 TPC-C 吞吐量。

从图中可以看出，吞吐量更加稳定。这种稳定性主要是由于在 BIOS 中禁用 NUMA，从而提高了内存访问效率并增强了整体系统稳定性。

然而，事务限流并非万能药，也有其局限性：

- 当最大数量的事务并发执行时，新事务必须等到现有事务完成。如果所有并发事务都由长时间运行的查询组成，可能会看起来好像 MySQL 系统停滞了 [31]。

值得注意的是，事务限流的实施细节及其灵活性是 AI 可以展示其实用性的领域。

## 8.4 缓解不断增长的性能下降

用户往往更容易注意到低并发性能的下降，而高并发性能的改进往往更难察觉。因此，保持低并发性能至关重要，因为它直接影响用户体验和升级意愿。

根据广泛的用户反馈，升级到 MySQL 8.0 后，用户普遍感到性能下降，特别是在批量插入和连接操作中。这种下降趋势在更高版本的 MySQL 中变得更加明显。此外，一些 MySQL 爱好者和测试人员报告了升级后多个 sysbench 测试中的性能退化。

这些性能问题能否避免？或者更具体地说，我们应该如何科学地评估性能持续下降的趋势？这些都是需要考虑的重要问题。

尽管官方团队持续优化，但性能的逐渐恶化不容忽视。在某些场景中，可能看起来有改进，但这并不意味着所有场景中的性能都得到了同等优化。此外，很容易以牺牲其他领域性能退化为代价来优化特定场景的性能。

### 8.4.1 MySQL 性能下降的根本原因

一般来说，随着添加更多功能，代码库增长，随着功能的不断扩展，性能变得越来越难以控制。

MySQL 开发人员经常无法注意到性能的下降，因为每次代码库的添加只会导致非常小的性能下降。然而，随着时间的推移，这些小的下降会累积起来，导致显著的累积效应，使用户感觉到较新版本的 MySQL 性能明显退化。

例如，下图显示了简单的单个连接操作的性能，MySQL 8.0.40 与 MySQL 8.0.27 相比显示出性能下降：

<img src="../media/image-join-degrade.png" alt="image-join-degrade" style="zoom:150%;" />

图 8-37. MySQL 8.0.40 中连接性能的显著下降。

下图显示了单并发下的批量插入性能测试，MySQL 8.0.40 与 5.7.44 版本相比的性能下降：

<img src="../media/image-bulk-insert-degrade.png" alt="image-bulk-insert-degrade" style="zoom:150%;" />

图 8-38. MySQL 8.0.40 中批量插入性能的显著下降。

从上面的两个图可以看出，8.0.40 版本的性能不好。

接下来，让我们从代码层面分析 MySQL 性能退化的根本原因。下面是 MySQL 8.0 中的 ***PT_insert_values_list::contextualize*** 函数：

```c++
bool PT_insert_values_list::contextualize(Parse_context *pc) {
  if (super::contextualize(pc)) return true;
  for (List_item *item_list : many_values) {
    for (auto it = item_list->begin(); it != item_list->end(); ++it) {
      if ((*it)->itemize(pc, &*it)) return true;
    }
  }

  return false;
}
```

MySQL 5.7 中相应的 ***PT_insert_values_list::contextualize*** 函数如下：

```c++
bool PT_insert_values_list::contextualize(Parse_context *pc)
{
  if (super::contextualize(pc))
    return true;
  List_iterator<List_item> it1(many_values);
  List<Item> *item_list;
  while ((item_list= it1++))
  {
    List_iterator<Item> it2(*item_list);
    Item *item;
    while ((item= it2++))
    {
      if (item->itemize(pc, &item))
        return true;
      it2.replace(item);
    }
  }

  return false;
}
```

从代码比较来看，MySQL 8.0 的代码看起来更优雅，似乎取得了进步。

不幸的是，很多时候正是这些代码改进背后的动机导致了性能退化。MySQL 官方团队用 ***deque*** 替换了之前的 ***List*** 数据结构，这已成为性能逐渐退化的根本原因之一。让我们看看 ***deque*** 文档：

```
std::deque (double-ended queue) is an indexed sequence container that allows fast insertion and deletion at both its
beginning and its end. In addition, insertion and deletion at either end of a deque never invalidates pointers or
references to the rest of the elements.

As opposed to std::vector, the elements of a deque are not stored contiguously: typical implementations use a sequence
of individually allocated fixed-size arrays, with additional bookkeeping, which means indexed access to deque must
perform two pointer dereferences, compared to vector's indexed access which performs only one.

The storage of a deque is automatically expanded and contracted as needed. Expansion of a deque is cheaper than the
expansion of a std::vector because it does not involve copying of the existing elements to a new memory location. On
the other hand, deques typically have large minimal memory cost; a deque holding just one element has to allocate its
full internal array (e.g. 8 times the object size on 64-bit libstdc++; 16 times the object size or 4096 bytes,
whichever is larger, on 64-bit libc++).

The complexity (efficiency) of common operations on deques is as follows:
Random access - constant O(1).
Insertion or removal of elements at the end or beginning - constant O(1).
Insertion or removal of elements - linear O(n).
```

如上述描述所示，在极端情况下，保留单个元素需要分配整个数组，导致内存效率非常低。例如，在批量插入中，需要插入大量记录时，官方实现将每条记录存储在单独的 deque 中。即使记录内容很少，仍然必须分配 deque。MySQL deque 实现为每个 deque 分配 1KB 的内存以支持快速查找。

```
The implementation is the same as classic std::deque: Elements are held in blocks of about 1 kB each.
```

官方实现使用 1KB 的内存来存储索引信息，即使记录长度不大但记录数量很多，内存访问地址可能变得不连续，导致缓存友好性较差。这种设计旨在改善缓存友好性，但并未完全有效。

值得注意的是，原始实现使用了 List 数据结构，其中内存通过内存池分配，提供了一定程度的缓存友好性。虽然随机访问效率较低，但优化对 List 元素的顺序访问会显著提高性能。

在升级到 MySQL 8.0 期间，用户观察到批量插入性能显著下降，主要原因之一是底层数据结构的重大变化。

此外，虽然官方团队改进了 redo log 机制，但这也导致了 MTR commit 操作效率的下降。与 MySQL 5.7 相比，添加的代码显著降低了单个提交的性能，尽管整体写入吞吐量已大大提高。

让我们检查 MySQL 5.7.44 中 MTR commit 的核心 ***execute*** 操作：

```c++
/** Write the redo log record, add dirty pages to the flush list and
release the resources. */
void mtr_t::Command::execute()
{
  ut_ad(m_impl->m_log_mode != MTR_LOG_NONE);
  if (const ulint len = prepare_write()) {
    finish_write(len);
  }
  if (m_impl->m_made_dirty) {
    log_flush_order_mutex_enter();
  }
  /* It is now safe to release the log mutex because the
  flush_order mutex will ensure that we are the first one
  to insert into the flush list. */
  log_mutex_exit();
  m_impl->m_mtr->m_commit_lsn = m_end_lsn;
  release_blocks();
  if (m_impl->m_made_dirty) {
    log_flush_order_mutex_exit();
  }
  release_all();
  release_resources();
}
```

让我们检查 MySQL 8.0.40 中 MTR commit 的核心 ***execute*** 操作：

```c++
/** Write the redo log record, add dirty pages to the flush list and
release the resources. */
void mtr_t::Command::execute() {
  ut_ad(m_impl->m_log_mode != MTR_LOG_NONE);
#ifndef UNIV_HOTBACKUP
  ulint len = prepare_write();
  if (len > 0) {
    mtr_write_log_t write_log;
    write_log.m_left_to_write = len;
    auto handle = log_buffer_reserve(*log_sys, len);
    write_log.m_handle = handle;
    write_log.m_lsn = handle.start_lsn;
    m_impl->m_log.for_each_block(write_log);
    ut_ad(write_log.m_left_to_write == 0);
    ut_ad(write_log.m_lsn == handle.end_lsn);
    log_wait_for_space_in_log_recent_closed(*log_sys, handle.start_lsn);
    DEBUG_SYNC_C("mtr_redo_before_add_dirty_blocks");
    add_dirty_blocks_to_flush_list(handle.start_lsn, handle.end_lsn);
    log_buffer_close(*log_sys, handle);
    m_impl->m_mtr->m_commit_lsn = handle.end_lsn;
  } else {
    DEBUG_SYNC_C("mtr_noredo_before_add_dirty_blocks");
    add_dirty_blocks_to_flush_list(0, 0);
  }
#endif /* !UNIV_HOTBACKUP */
```

通过比较，很明显在 MySQL 8.0.40 中，MTR commit 中的 execute 操作变得复杂得多，涉及更多步骤。这种复杂性是低并发写入性能下降的主要原因之一。

特别是，操作 ***m_impl-\>m_log.for_each_block(write_log)*** 和 ***log_wait_for_space_in_log_recent_closed(*log_sys, handle.start_lsn)*** 有显著的开销。这些更改是为了增强高并发性能而做的，但代价是低并发性能。

redo log 对高并发模式的优先导致低并发工作负载的性能较差。尽管引入 ***innodb_log_writer_threads*** 旨在缓解低并发性能问题，但它不影响上述函数的执行。由于这些操作变得更加复杂并需要频繁的 MTR 提交，性能仍然显著下降。

让我们看看 instant add/drop 功能对性能的影响。下面是 MySQL 5.7 中的 ***rec_init_offsets_comp_ordinary*** 函数：

```c++
/******************************************************//**
Determine the offset to each field in a leaf-page record
in ROW_FORMAT=COMPACT.  This is a special case of
rec_init_offsets() and rec_get_offsets_func(). */
UNIV_INLINE MY_ATTRIBUTE((nonnull))
void
rec_init_offsets_comp_ordinary(
/*===========================*/
  const rec_t*    rec,  /*!< in: physical record in
          ROW_FORMAT=COMPACT */
  bool      temp, /*!< in: whether to use the
          format for temporary files in
          index creation */
  const dict_index_t* index,  /*!< in: record descriptor */
  ulint*      offsets)/*!< in/out: array of offsets;
          in: n=rec_offs_n_fields(offsets) */
{
  ulint   i   = 0;
  ulint   offs    = 0;
  ulint   any_ext   = 0;
  ulint   n_null    = index->n_nullable;
  const byte* nulls   = temp
    ? rec - 1
    : rec - (1 + REC_N_NEW_EXTRA_BYTES);
  const byte* lens    = nulls - UT_BITS_IN_BYTES(n_null);
  ulint   null_mask = 1;

  ...
  ut_ad(temp || dict_table_is_comp(index->table));

  if (temp && dict_table_is_comp(index->table)) {
    /* No need to do adjust fixed_len=0. We only need to
    adjust it for ROW_FORMAT=REDUNDANT. */
    temp = false;
  }
  /* read the lengths of fields 0..n */
  do {
    const dict_field_t* field
      = dict_index_get_nth_field(index, i);
    const dict_col_t* col
      = dict_field_get_col(field);
    ulint     len;
  ...
```

MySQL 8.0.40 中的 ***rec_init_offsets_comp_ordinary*** 函数如下：

```c++
/** Determine the offset to each field in a leaf-page record in
ROW_FORMAT=COMPACT.  This is a special case of rec_init_offsets() and
rec_get_offsets().
...
*/
inline void rec_init_offsets_comp_ordinary(const rec_t *rec, bool temp,
                                           const dict_index_t *index,
                                           ulint *offsets) {
  ...
  const byte *nulls = nullptr;
  const byte *lens = nullptr;
  uint16_t n_null = 0;
  enum REC_INSERT_STATE rec_insert_state = REC_INSERT_STATE::NONE;
  uint8_t row_version = UINT8_UNDEFINED;
  uint16_t non_default_fields = 0;

  if (temp) {
    rec_insert_state = rec_init_null_and_len_temp(
        rec, index, &nulls, &lens, &n_null, non_default_fields, row_version);
  } else {
    rec_insert_state = rec_init_null_and_len_comp(
        rec, index, &nulls, &lens, &n_null, non_default_fields, row_version);
  }

  ut_ad(temp || dict_table_is_comp(index->table));
  if (temp) {
    if (dict_table_is_comp(index->table)) {
      /* No need to do adjust fixed_len=0. We only need to
      adjust it for ROW_FORMAT=REDUNDANT. */
      temp = false;
    } else {
      /* Redundant temp row. Old instant record is logged as version 0.*/
      if (rec_insert_state == INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION ||
          rec_insert_state == INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION) {
        rec_insert_state = INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION;
        ut_ad(row_version == UINT8_UNDEFINED);
      }
    }
  }

  /* read the lengths of fields 0..n */
  ulint offs = 0;
  ulint any_ext = 0;
  ulint null_mask = 1;
  uint16_t i = 0;
  do {
    /* Fields are stored on disk in version they are added in and are
    maintained in fields_array in the same order. Get the right field. */
    const dict_field_t *field = index->get_physical_field(i);
    const dict_col_t *col = field->col;
    uint64_t len;

    switch (rec_insert_state) {
      case INSERTED_INTO_TABLE_WITH_NO_INSTANT_NO_VERSION:
        ut_ad(!index->has_instant_cols_or_row_versions());
        break;

      case INSERTED_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION: {
        ut_ad(row_version == UINT8_UNDEFINED || row_version == 0);
        ut_ad(index->has_row_versions() || temp);
        /* Record has to be interpreted in v0. */
        row_version = 0;
      }
        [[fallthrough]];
      case INSERTED_AFTER_UPGRADE_BEFORE_INSTANT_ADD_NEW_IMPLEMENTATION:
      case INSERTED_AFTER_INSTANT_ADD_NEW_IMPLEMENTATION: {
        ...
      } break;
      case INSERTED_BEFORE_INSTANT_ADD_OLD_IMPLEMENTATION:
      case INSERTED_AFTER_INSTANT_ADD_OLD_IMPLEMENTATION: {
        ...
      } break;

      default:
        ut_ad(false);
    }
    ...
```

从上面的代码可以清楚地看出，随着 instant add/drop column 功能的引入，***rec_init_offsets_comp_ordinary*** 函数变得明显更加复杂，引入了更多的函数调用并添加了严重影响缓存优化的 switch 语句。由于这个函数被频繁调用，它直接影响了更新索引、批量插入和连接的性能，导致主要的性能损失。

此外，MySQL 8.0 中的性能下降不仅限于上述内容；还有许多其他领域导致整体性能退化，特别是内联函数扩展的影响。例如，以下代码影响内联函数的扩展：

```c++
void validate_rec_offset(const dict_index_t *index, const ulint *offsets,
                         ulint n, ut::Location L) {
  ut_ad(rec_offs_validate(nullptr, nullptr, offsets));
  if (n >= rec_offs_n_fields(offsets)) {
#ifndef UNIV_NO_ERR_MSGS
    dump_metadata_dict_table(index->table);
    auto num_fields = static_cast<size_t>(rec_offs_n_fields(offsets));
    ib::fatal(L, ER_IB_DICT_INVALID_COLUMN_POSITION, ulonglong{n}, num_fields);
#endif /* !UNIV_NO_ERR_MSGS */
  }
}
```

根据我们的测试，***ib::fatal*** 语句严重干扰了内联优化。对于频繁访问的函数，建议避免干扰内联优化的语句。

接下来，让我们看一个类似的问题。***row_sel_store_mysql_field*** 函数被频繁调用，其中 ***row_sel_field_store_in_mysql_format*** 是其中的热点函数。具体代码如下：

```c++
// clang-format off
/** Convert a field in the Innobase format to a field in the MySQL format.
@param[out]     mysql_rec       Record in the MySQL format
@param[in,out]  prebuilt        Prebuilt struct
@param[in]      rec             InnoDB record; must be protected by a page
                                latch
@param[in]      rec_index       Index of rec
@param[in]      prebuilt_index  prebuilt->index
@param[in]      offsets         Array returned by rec_get_offsets()
@param[in]      field_no        templ->rec_field_no or
                                templ->clust_rec_field_no or
                                templ->icp_rec_field_no or sec field no if
                                clust_templ_for_sec is true
@param[in]      templ           row template
@param[in]      sec_field_no    Secondary index field no if the secondary index
                                record but the prebuilt template is in
                                clustered index format and used only for end
                                range comparison.
@param[in]      lob_undo        the LOB undo information.
@param[in,out]  blob_heap       If not null then use this heap for BLOBs */
// clang-format on
[[nodiscard]] static bool row_sel_store_mysql_field(
    byte *mysql_rec, row_prebuilt_t *prebuilt, const rec_t *rec,
    const dict_index_t *rec_index, const dict_index_t *prebuilt_index,
    const ulint *offsets, ulint field_no, const mysql_row_templ_t *templ,
    ulint sec_field_no, lob::undo_vers_t *lob_undo, mem_heap_t *&blob_heap) {
  DBUG_TRACE;
  ...
    } else {
    /* Field is stored in the row. */

    data = rec_get_nth_field_instant(rec, offsets, field_no, index_used, &len);

    if (len == UNIV_SQL_NULL) {
      /* MySQL assumes that the field for an SQL
      NULL value is set to the default value. */
      ut_ad(templ->mysql_null_bit_mask);

      UNIV_MEM_ASSERT_RW(prebuilt->default_rec + templ->mysql_col_offset,
                         templ->mysql_col_len);
      mysql_rec[templ->mysql_null_byte_offset] |=
          (byte)templ->mysql_null_bit_mask;
      memcpy(mysql_rec + templ->mysql_col_offset,
             (const byte *)prebuilt->default_rec + templ->mysql_col_offset,
             templ->mysql_col_len);
      return true;
    }

    if (DATA_LARGE_MTYPE(templ->type) || DATA_GEOMETRY_MTYPE(templ->type)) {
      ...
      mem_heap_t *heap{};

      if (blob_heap == nullptr) {
        blob_heap = mem_heap_create(UNIV_PAGE_SIZE, UT_LOCATION_HERE);
      }

      heap = blob_heap;
      data = static_cast<byte *>(mem_heap_dup(heap, data, len));
    }

    /* Reassign the clustered index field no. */
    if (clust_templ_for_sec) {
      field_no = clust_field_no;
    }

    row_sel_field_store_in_mysql_format(mysql_rec + templ->mysql_col_offset,
                                        templ, rec_index, field_no, data, len,
                                        sec_field_no);
    ...
```

***row_sel_field_store_in_mysql_format*** 函数最终调用 ***row_sel_field_store_in_mysql_format_func***。

```c++
/** Convert a non-SQL-NULL field from Innobase format to MySQL format. */
static inline void row_sel_field_store_in_mysql_format(
    byte *dest, const mysql_row_templ_t *templ, const dict_index_t *idx,
    ulint field, const byte *src, ulint len, ulint sec) {
  row_sel_field_store_in_mysql_format_func(
      dest, templ, idx, IF_DEBUG(field, ) src, len IF_DEBUG(, sec));
}
```

由于存在 ***ib::fatal*** 代码，***row_sel_field_store_in_mysql_format_func*** 函数无法内联。

```c++
void row_sel_field_store_in_mysql_format_func(
    byte *dest, const mysql_row_templ_t *templ, const dict_index_t *index,
    IF_DEBUG(ulint field_no, ) const byte *data,
    ulint len IF_DEBUG(, ulint sec_field)) {
  byte *ptr;

  ...

  if (templ->is_multi_val) {
    ib::fatal(UT_LOCATION_HERE, ER_CONVERT_MULTI_VALUE)
        << "Table name: " << index->table->name
        << " Index name: " << index->name;
  }
```

频繁调用的低效函数，每秒执行数千万次，会严重影响连接性能。

让我们继续探索性能下降的原因。以下官方性能优化实际上是连接性能下降的根本原因之一。尽管某些查询可能得到改进，但它仍然是普通连接操作性能退化的原因之一。

```c++
commit ffe1726f2542505e486c4bcd516c30f36c8ed5f6 (HEAD)
Author: Knut Anders Hatlen <knut.hatlen@oracle.com>
Date:   Wed Dec 21 14:29:02 2022 +0100

    Bug#34891365: MySQL 8.0.29+ is slower than MySQL 8.0.28 with
    queries using JOINS

    The one-row cache in EQRefIterator was disabled for queries using
    streaming aggregation in the fix for bug#33674441. This was done in
    order to fix a number of wrong result bugs, but it turned out to have
    too big impact on the performance of many queries.

    This patch re-enables the cache for the above queries, and fixes the
    original wrong result bugs in a different way. There were two
    underlying problems that caused the wrong results:

    1) AggregateIterator::Init() does not restore the table buffers to the
    state they had after the last read from the child iterator in the
    previous invocation. The table buffers would have the content from the
    first row in the last group that was returned from the iterator in the
    previous invocation, rather than the contents of the last row read by
    the child iterator, and this made the cache in EQRefIterator return
    wrong values. Fixed by restoring the table buffers in
    AggregateIterator::Init() if the previous invocation had modified
    them.

    2) When the inner tables of an outer join are null-complemented, the
    table buffers are modified to contain NULL values, thereby disturbing
    the cached value for any EQRefIterator reading from one of the
    null-complemented tables. Fixed by making StoreFromTableBuffers()
    store the actual values contained in the table buffer instead of the
    null values, if the table is accessed by EQRefIterator.
    LoadIntoTableBuffers() is taught to restore those values, but
    additionally set the null flags on the columns after restoring them.

    The hypergraph optimizer had another workaround for these wrong
    results (it added a streaming step right before the aggregation). This
    workaround is also removed now.

    Change-Id: I554a90213cae60749dd6407e48a745bc71578e0c
```

MySQL 的问题不止于此。如上面的分析所示，MySQL 的性能下降不是没有原因的。一系列小问题累积起来，可能导致用户体验到明显的性能退化。然而，这些问题通常难以识别，使它们更难解决。

所谓的"过早优化是万恶之源"并不适用于 MySQL 开发。数据库开发是一个复杂的过程，随着时间的推移忽视性能会使后续的性能改进变得更加困难。

### 8.4.2 缓解 MySQL 性能下降的解决方案

写入性能下降的主要原因与 MTR commit 问题、instant add/drop column 以及其他几个因素有关。这些难以通过传统方式优化。然而，用户可以通过 PGO 优化来补偿性能下降。采用适当的策略，性能通常可以保持稳定。

对于批量插入性能退化，我们的开源版本 [64] 用改进的 list 实现替换了官方的 deque。这主要解决了内存效率问题，并可以部分缓解性能下降。通过将 PGO 优化与我们的开源版本结合使用，批量插入性能可以接近 MySQL 5.7。

<img src="../media/image-bulk-insert-optimize.png" alt="image-bulk-insert-optimize.png" style="zoom:150%;" />

图 8-39. 使用 PGO 优化的 MySQL 8.0.40 性能与 5.7 版本大致相当。

用户还可以利用多线程进行并发批处理，充分利用 redo log 改进的并发性，这可以显著提升批量插入性能。

关于更新索引问题，由于不可避免地添加新代码，PGO 优化可以帮助缓解这个问题。我们的 PGO 版本 [64] 可以显著缓解这个问题。

对于读取性能，特别是连接性能，我们进行了重大改进，包括修复内联问题和进行其他优化。加上 PGO，连接性能可以比官方版本提高 30% 以上。

<img src="../media/image-join-improve.png" alt="image-join-improve.png" style="zoom:150%;" />

图 8-40. 使用 PGO 的连接性能优化带来了显著改进。

我们将继续投入时间优化低并发性能。这个过程很长，但涉及许多需要改进的领域。

开源版本可供测试，我们将持续努力改进 MySQL 性能。

[下一章](Chapter9.md)
