# 附录

## 术语表

书中术语较多，先熟悉含义有助于阅读。

**1 在读写操作中实现强一致性**

组复制可用两种机制保证强一致：

**强一致读（Before）**：

读写（RW）事务在应用前等待此前事务完成；只读（RO）在执行前等待此前事务。这样读到的是足够新的值，主要增加读延迟。

**强一致写（After）**：

RW 事务要等变更应用到全部成员后才算完成，从而在本地提交后，其他成员上的后续读能看到该提交值或更新值；RO 不受影响。

**2 AI**

人工智能（AI）指让机器具备感知、学习与目标导向行为的方法与系统。机器学习（ML）是其子集：从数据中学习完成任务，而少依赖显式规则。神经网络进步带动了多个领域的效果提升。

深度学习是 ML 的重要分支，受益于算力（尤其 GPU）与大规模数据（如 ImageNet），在视觉、语音、自然语言等方向进展显著。

在数据库中，AI 可用于自动化运维、参数调优与 SQL 生成等。MySQL HeatWave 把事务、分析与机器学习放进同一托管服务。

**3 算法**

算法是解决特定问题的有穷步骤。设计算法依赖逻辑推理：要组织好控制流、数据流与交互，保证正确、高效并满足约束。

**4 平衡回放速度**

对从库回放而言，平衡回放速度是正常条件下从库刚好跟得上主库的速度阈值。主库不超过该阈值时，从库几乎无延迟；超过后，从库回放进度开始落后。

**5 因果关系**

若事件/状态 A（因）引起 B（果），则称二者有因果关系：果依赖因，因对果负部分责任。

**6 崩溃恢复**

事务可能中途失败。若提交前发生故障，库可能不一致甚至不可用。崩溃恢复就是把库拉回一致、可用状态的过程。

**7 数据结构**

数据结构是组织与存储数据的方式，通常为高效访问而选。它包含数据值、关系与可执行操作，构成代数结构。

**8 双写**

双写缓冲区是 InnoDB 在把页写入数据文件最终位置前，先把缓冲池页写入的中间区域。写页过程中意外故障时，崩溃恢复可用该副本恢复页面。

**9 双一**

Binlog 是记录事务逻辑修改的日志，用于复制与容灾；redo log 是记录数据页物理修改的日志，用于崩溃后恢复已提交数据。二者对崩溃恢复都关键。为防故障丢数据，需保证 binlog 与 redo 及时落盘，即常说的“双一”。

**10 容错**

容错指系统在部分组件故障或出错时仍能正确继续工作的能力。

**11 GTID**

全局事务标识符（GTID）为每个事务赋予唯一 ID，简化复制：搭建从库或故障转移时不必再死盯日志文件与位点。基于 GTID 时，若主库已提交事务都出现在从库，即可判定一致。

GTID 在主从间维护，便于沿 binlog 追溯事务来源。某 GTID 一旦在服务器上提交，重复到达会被忽略，从库只应用一次。

本书测试前通常开启：

```
gtid_mode=ON
enforce_gtid_consistency=ON
```

使各节点统一走 GTID，便于跟踪与排障。

**12 高可用性**

高可用指系统在故障与运维扰动下，仍能维持可接受服务水平的能力。

**13 幂等性**

幂等指某操作执行多次与执行一次效果相同。

**14 Latch 与 Lock：关键区别**

Latch / mutex 是同步原语，防止多线程同时碰同一状态。Lock（锁）则保护数据库数据的并发访问，保证结果一致。

Latch 更接近 mutex；MySQL 的锁结构示意如下：

```c++
/** 锁结构；由 lock_sys latches 保护 */
struct lock_t {
  /** 拥有该锁的事务 */
  trx_t *trx;
  /** 事务的锁列表 */
  UT_LIST_NODE_T(lock_t) trx_locks;
  /** 记录锁的索引 */
  dict_index_t *index;
  /** 记录锁的哈希链节点。单链表中的链接节点，由哈希表使用。*/
  lock_t *hash;
  union {
    /** 表锁 */
    lock_table_t tab_lock;
    /** 记录锁 */
    lock_rec_t rec_lock;
  };
  ...
```

二者差别可用比喻说明 [19]：

- **latch**：把门窗“扣住”以免晃动，但不等于防盗。
- **lock**：没钥匙进不来，强调访问控制。

MySQL 里全局 latch 用来串行化某些处理。例如：

```c++
除了上述所有步骤（除了步骤 2，因为我们通常已经知道页面）都是通过单行完成的：
    locksys::Shard_latches_guard guard{*block_a, *block_b};
要"停止世界"，只需使用以下方式对全局 latch 进行 x-latch：
    locksys::Global_exclusive_latch_guard guard{};
这个类不暴露太多公共函数，因为意图是使用友元守卫类，如演示的 Shard_latches_guard。
*/
class Latches {
 private:
  using Lock_mutex = ib_mutex_t;
 ...
```

锁属于事务模型，常见有行锁、表锁；死锁检测针对的是 lock，不是 latch。

理解锁有助于：

- 做大规模、高负载或高可靠应用
- 做性能调优

熟悉 InnoDB 锁与事务模型很有必要。

另需注意：锁对象本身也要靠 latch 保护正确性，例如：

```c++
/** 为等待的锁请求授予锁并释放等待的事务。调用者必须持有包含该锁的分片的 lock_sys latch，
但不持有 lock->trx->mutex。
@param[in,out]    lock    等待的锁请求
 */
static void lock_grant(lock_t *lock) {
  ut_ad(locksys::owns_lock_shard(lock));
  ut_ad(!trx_mutex_own(lock->trx));
  trx_mutex_enter(lock->trx);
  if (lock_get_mode(lock) == LOCK_AUTO_INC) {
    dict_table_t *table = lock->tab_lock.table;
    if (table->autoinc_trx == lock->trx) {
      ib::error(ER_IB_MSG_637) << "事务已经拥有"
                               << " AUTO-INC 锁！";
    } else {
      ut_ad(table->autoinc_trx == nullptr);
      table->autoinc_trx = lock->trx;
      ib_vector_push(lock->trx->lock.autoinc_locks, &lock);
    }
  }
  ...
```

**15 使用 replica_preserve_commit_order 维护事务顺序**

*replica_preserve_commit_order* 要求从库按中继日志中的顺序提交事务，从而保持因果：主库上 A 先于 B 提交，从库上 A 也先于 B，避免从库上读到颠倒顺序。

**16 MVCC**

多版本并发控制（MVCC）是一种尽量少用锁的并发控制手段，支持多事务并发访问。

**17 网络延迟**

包交换网络中延迟常按往返时延（RTT）衡量，对 MySQL 性能影响很大。

**18 网络分区**

网络被切成彼此隔离的子网，可能是设计使然，也可能是故障。分布式软件需具备分区容错：分区发生时仍尽量正确运行。

**19 NUMA**

非均匀内存访问（NUMA）下，访存时延取决于内存相对处理器的位置。它改善了 SMP 的扩展瓶颈（SMP 易在共享内存路径上堵死）。今日服务器多为 NUMA；若把 MySQL 绑到单 NUMA 节点，行为上又接近纯 SMP。

**20 OLTP**

在线事务处理（OLTP）面向实时业务请求；相对地，OLAP 侧重分析而非高频事务。

**21 Paxos 算法**

Paxos 是一族在不可靠/易错处理器网络中解决共识的协议。

**22 流水线**

流水线类似装配线：各阶段可重叠推进，即便存在先后依赖，从而提高吞吐、缩短总耗时。

**23 PGO**

配置文件引导优化（PGO）用运行期 profile 指导编译，改进热点路径。

**24 读已提交隔离级别**

读已提交保证读到的是已提交数据，避免脏读；但不保证同一事务内再次读同一数据仍相同——读后数据仍可能被别人改掉。

**25 复制**

主库把库结构/内容变更记入 binlog（更新、删除等；SELECT 不记），从库读取并应用这些事件。事件格式随类型而变。

**26 响应时间**

响应时间是系统应答一次服务请求所花的时间，衡量“有多快响应”。

**27 基于行的复制**

基于行的日志下，主库把逐行变更写入 binlog，再复制到从库应用，即基于行的复制。

**28 状态机复制**

状态机复制（SMR）通过复制服务端并协调客户端与副本交互实现容错。MySQL 组复制用 Paxos 做 SMR。

**29 线程池**

线程池预先准备一批工作线程接任务，避免频繁创建/销毁线程；池大小按算力资源调节。

**30 吞吐量**

吞吐量是单位时间处理的请求量。常见指标：

1. **TPS**：每秒事务数
2. **QPS**：每秒查询数
3. **tpmC**：TPC-C 中 New-Order 每分钟事务率
4. **tpmTOTAL**：TPC-C 总事务每分钟率

**31 惊群**

许多进程/线程被同一事件唤醒，却只有一个能处理，造成无效争抢，严重时系统像“卡死”。

**32 TPC-C**

TPC-C 是 OLTP 基准，主指标是 New-Order 的每分钟事务率 tpmC。

**33 事务**

事务是逻辑工作单元，常含多步操作；既简化应用编码，也是 DBMS 核心能力。

**34 事务限流**

通过限制同时进入事务系统的用户线程数控制高并发负载，减轻压力、提高稳定性。

**35 视图变更**

视图表示某一时刻组复制中的活跃成员集合。成员加入/离开时发生视图变更，并同步通知所有成员；每次变更生成唯一视图标识。

## **测试工具**

### 1 SysBench

SysBench 是常用的开源 DBMS 压测工具，上手快，支持读写/只写，以及 Pareto 等高冲突场景。

优点是简单；缺点也在简单：过度简化可能失真，漏掉关键问题，不能代表真实在线能力。它适合快速摸底，不宜单独当作“线上能力”的充分证明。

### 2 TPC-C 测试工具

TPC-C（事务处理性能委员会）是经典 OLTP 基准：9 张表、10 个外键；除 Item 外，表规模随仓库数 W 缩放。

<img src="../media/3b2c3f6b25df292e0ed14955df4a7328.png" style="zoom:50%;" />

此模式由五个不同的事务使用，每个事务创建不同的访问模式：

1. **Item**：只读。
2. **Warehouse、District、Customer、Stock**：读写。
3. **New-Order**：插入、读取和删除。
4. **Order 和 Order-Line**：具有时间延迟更新的插入，导致行变旧且不经常读取。
5. **History**：仅插入。

事务种类不多但访问模式多样，使 TPC-C 长期保持重要性。本书主要用 BenchmarkSQL [68] 测 MySQL 的 TPC-C。

## MySQL 如何处理 SQL？

请求-响应是基本网络通信模型：一端发请求，另一端处理后回复。

MySQL 亦然：客户端发 SQL，Server 处理后返回。MySQL Server 8.0 的标准处理流程如下：

![](../media/f3b8490d4e41a8da5e954aabf0a4aacf.png)

以下是 MySQL Server 如何通过详细示例处理 SQL 请求的方法。假设用户从 MySQL 客户端向 MySQL Server 发送以下 SQL 语句：

```
select name from student where id=1;
```

在执行 SQL 查询之前，MySQL Server 首先使用"解析器"解析 SQL 语句，该解析器执行两个基本任务：

**1. 词法分析**

MySQL Server 扫描您输入的 SQL 字符串并将其转换为 token，识别关键字和其他元素。

![](../media/b9bd3a0655d66dfdb5690e999db6ac32.png)

**2. 语法分析**

使用词法分析中的 token，语法解析器检查 SQL 语句是否符合 MySQL 语法规则。验证成功后，它构建一个 SQL 语法树。这个树结构帮助后续模块从 WHERE 子句中提取关键组件，如 SQL 类型（例如 SELECT、INSERT）、表名、字段名和条件。例如，提供的 SQL 语句将生成表示这些组件的语法树：

![](../media/0e32e448841daf774e632cb0884cd7f4.png)

这捕获了解析过程的核心，MySQL Server 使用 Bison 解析器构建语法树。

解析后，MySQL Server 在实际执行开始之前执行几个步骤来优化查询性能：

**1. 预处理器**

预处理器执行初步任务，例如验证表或字段的存在，并扩展通配符，如 \`select \*\` 中的 \`\*\` 以包含所有表列。

**2. 查询优化器**

查询优化器确定 SQL 查询的执行计划。此阶段包括：

- **逻辑查询重写**：将查询转换为逻辑等效形式。
- **基于成本的连接优化**：评估不同的连接方法以最小化执行成本。
- **基于规则的访问路径选择**：根据预定义规则选择最佳数据访问路径。

查询优化器生成执行计划，然后由查询执行引擎使用。

完成执行计划后，查询执行引擎开始执行 SQL 语句，逐记录与存储引擎交互。

以下是两种类型的执行过程：全表扫描和索引查询。

**1. 全表扫描**

假设进行查询以检索所有年龄大于 20 岁的学生信息：

```
select * from student where age > 20;
```

由于此查询条件不使用索引，优化器通过将访问类型设置为 ALL 来选择全表扫描。

![](../media/71732b73a7cb4fe508f504fca85b0db3.png)

执行器和存储引擎的执行过程如下：

1. Server 层调用存储引擎的全扫描接口开始从表中读取记录。
2. 执行器检查检索到的记录的年龄是否超过 20。如果有可用空间，满足此条件的记录将被分派到网络写入缓冲区。
3. 执行器循环从存储引擎请求下一条记录。根据查询条件评估每条记录，满足条件的记录将被发送到网络写入缓冲区，前提是缓冲区未满。
4. 一旦存储引擎从表中读取了所有记录，它将通知执行器读取完成。
5. 收到完成信号后，执行器退出循环并将查询结果刷新到客户端。

为了优化性能，MySQL 通过在将记录发送到客户端之前检查网络缓冲区是否已满来最小化频繁的写系统调用。仅当缓冲区已满或收到完成信号时才发送记录。

**2. 索引查询**

考虑以下 SQL 查询：

```
select name from student where id < 3 and name like 'wang%';
```

在执行此查询之前，在 student 表的 name 字段上添加辅助索引：

```
alter table student add index index_name(name);
```

可以使用 *'explain'* 语句查看此 SQL 查询的执行计划，该语句显示查询现在利用了新创建的索引。

![](../media/f0839a5643114a83f16d647c8c8299d3.png)

使用索引的执行过程如下：

1. 执行器请求存储引擎定位与查询条件匹配的第一个索引记录（例如 name LIKE 'wang%'）。

2. 存储引擎检索并将匹配的索引记录返回给 Server 层。

3. 执行器检查记录是否满足额外的查询条件（例如 id \< 3）。

   如果满足条件，除非缓冲区已满，否则将相应的 name 添加到网络缓冲区。如果不满足条件，执行器跳过该记录并从存储引擎请求下一条记录。

4. 此循环继续，执行器反复请求并评估与查询条件匹配的下一个索引记录，直到处理完所有相关索引记录。

5. 一旦存储引擎指示所有相关索引记录已被处理，执行器退出循环并将收集的结果发送到客户端。

使用索引允许存储引擎快速定位必要的记录，绕过扫描整个表的需要。通常，这明显提高了执行效率并加快了查询速度。

## MySQL 架构

下图说明了客户端-服务器架构：

![](../media/fd0d20c24fda6e6b4d654a468ed84d2c.png)

MySQL 遵循客户端-服务器架构，将系统分为两个主要组件：客户端和服务器。

### 1 客户端

1. 客户端是与 MySQL 数据库服务器交互的应用程序。
2. 它可以是独立应用程序、Web 应用程序或任何需要数据库的程序。
3. 客户端将 SQL 查询发送到 MySQL 服务器进行处理。

### 2 服务器

1. 服务器是负责存储、管理和处理数据的 MySQL 数据库管理系统。
2. 它接收 SQL 查询，处理它们并返回结果集。
3. 它管理多个客户端的数据存储、安全性和并发访问。

客户端使用 MySQL 协议通过网络与服务器通信，使多个客户端能够同时交互。应用程序使用 MySQL 连接器连接到数据库服务器。MySQL 还提供客户端工具，例如基于终端的 MySQL 客户端，用于与服务器直接交互。

MySQL 数据库服务器包括几个守护进程：

1. **SQL 接口**：为应用程序提供标准化接口，使用 SQL 查询与数据库交互。
2. **查询解析器**：分析 SQL 查询以了解其结构和语法，将它们分解为组件以进行进一步处理。
3. **查询优化器**：评估给定查询的各种执行计划，并选择最有效的计划以提高性能。

在 MySQL 中，存储引擎负责数据的存储、检索和管理。MySQL 的可插拔存储引擎架构允许选择不同的存储引擎，例如 InnoDB 和 MyISAM，以满足特定的性能和可扩展性要求，同时保持一致的 SQL 接口。

文件系统组织和存储各种文件类型，包括数据和索引文件。MySQL 使用日志文件，如二进制日志和 redo 日志，来维护事务一致性并支持恢复机制。

总体来看，MySQL 遵循客户端-服务器架构，其中客户端将 SQL 查询发送到服务器进行处理。MySQL 支持可插拔存储引擎架构，允许以各种特性和性能特征管理数据存储和检索。

## MySQL 集群

创建容错系统的最常见方法是使用冗余组件，允许系统在一个组件发生故障时继续运行。

MySQL 中的复制将数据从一台服务器（主库）复制到一台或多台服务器（从库），提供了几个优点：

1. **横向扩展解决方案**：在多个从库之间分散负载以提高性能。所有写入和更新都发生在主库上，而读取可以发生在从库上，从而提高读取速度。
2. **分析**：允许在从库上进行分析而不影响主库性能。
3. **长距离数据分发**：为远程站点创建本地数据副本，而无需持续访问主库。

原始同步类型是单向异步复制。异步复制的优点是用户响应时间不受从库影响。但是，如果主库发生故障且从库未完全同步，则存在数据丢失的重大风险。

半同步复制除了异步复制之外，还要求主库上的提交等待至少一个从库确认并记录事务事件。这确保了从库上的数据是最新的，但影响用户响应时间并引入高可用性复杂性。

复制引入了显著的复杂性，因为它需要管理多个服务器而不是一个。这涉及解决经典的分布式系统问题，如网络分区和脑裂场景。主要挑战是一致地协调这些服务器，确保它们就系统和数据状态达成一致，并随着每次更改保持一致。本质上，服务器必须作为分布式状态机运行，要么作为单个实体前进，要么最终收敛到相同的状态。

MySQL 组复制通过强大的服务器协调提供分布式状态机复制。组内的服务器通过内置的成员服务自动维护一致的视图，在服务器加入或离开时进行更新。如果服务器发生故障，故障检测机制会提醒组。

事务提交需要对全局事务序列达成多数共识，确保统一的提交或中止决策。在脑裂场景中，网络分区会停止进度，直到问题解决。组通信系统（GCS）协议通过故障检测、成员管理和有序消息传递确保一致的数据复制，所有这些都由 Paxos 算法作为核心通信引擎提供支持。

MySQL 组复制可以在单主模式下运行，具有自动主选举功能，一次只允许一台服务器接受更新。或者，高级用户可以在多主模式下部署组，其中所有服务器都可以接受并发更新。但是，这要求应用程序管理此类部署的限制。

通常，更简单的集群设置更容易配置，但在出现问题时可能会变得更具挑战性。相比之下，更复杂的集群最初更难配置，但在处理问题时提供了更优雅的解决方案。复制机制的选择应基于具体的用例。

## 测试相关材料

这里提供了详细的硬件配置、操作系统和各种测试脚本，以帮助读者复制本书中提供的一些测试结果。

### 1 硬件配置

Intel(R) Xeon(R) Gold 6238 CPU @ 2.10GHz

cache_alignment : 64

cpu MHz : 3700.000

cache size : 30976 KB

x86 架构上 NUMA 环境中的节点详细信息：

![](../media/d5e1fe52debddd4c5b288e3b6e9c5262.png)

磁盘都是 NVMe SSD。

除非另有说明，否则大多数测试都在这些硬件条件下进行。

### 2 操作系统

操作系统内核版本是 Linux 5.19.8。

### 3 BenchmarkSQL 脚本

创建 TPC-C 表的脚本可以在以下地址找到：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/tableCreates.sql_base_for_test

创建相关索引的脚本可以在以下地址找到：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/indexCreates.sql_base_for_test

BenchmarkSQL 测试的主要配置脚本如下：

```
warehouses=1000
loadWorkers=100
terminals=300
warehouses-begin=1
warehouses-end=1000
//要运行每个终端的指定事务数 - runMins 必须等于零
runTxnsPerTerminal=0
//要运行指定的分钟数 - runTxnsPerTerminal 必须等于零
runMins=5
//每分钟的总事务数
limitTxnsPerMin=0
//设置为 true 以在 4.x 兼容模式下运行。设置为 false 以均匀使用
//整个配置的数据库。
terminalWarehouseFixed=true
//以下五个值必须加起来等于 100
//默认百分比 45、43、4、4 和 4 与 TPC-C 规范匹配
newOrderWeight=45
paymentWeight=43
orderStatusWeight=4
deliveryWeight=4
stockLevelWeight=4
```

如图所示，通常有 1000 个仓库，每次测试持续 5 分钟。

### 4 SysBench 脚本

SysBench 测试非常简单；这里只是关键参数。

SysBench 测试参数：*-table_size=1000000 --tables=1*

对于低争用的测试，使用参数 *--rand-type=uniform*；对于高争用的测试，使用参数 *--rand-type=pareto*。

### 5 使用 tpcc-mysql 脚本进行基准测试

tpcc-mysql 改进工具的下载链接：http://www.anysql.net/

![](../media/77a84346325aa692f5e8b92741c6efb9.png)

如何使用 tpcc-mysql 工具？

假设用户名是 xxx，密码是 yyy，步骤如下：

数据加载命令如下：

```
./tpcc_load -h127.0.0.1 -d tpcc200 -u xxx -p "yyy" -P 3306 -w 200
```

测试命令如下：

```
./tpcc_start -h127.0.0.1 -P 3306 -d tpcc200 -u xxx -p "yyy" -w 200 -c 100 -r 0 -l 60 -F 1
```

### 6 配置参数

由于测试众多，这里仅列出典型配置。特殊配置需要相应的参数修改。

有关独立 MySQL 实例的典型配置参数的具体详细信息，请参阅以下地址：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/my.cnf_base_for_test

需要注意的是，默认情况下，测试在 *Read Committed* 事务隔离级别下进行，启用了 *binary logging*、*'dual one'* 配置和 *doublewrite*。

对于组复制，原生 MySQL 版本中的主库配置参数如下：

```
# for mgr
plugin_load_add='group_replication.so'
enforce-gtid-consistency
gtid-mode=on
loose-group_replication_member_expel_timeout=3
loose-group_replication_start_on_boot= OFF
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-baaaaaaaaaab"
loose-group_replication_local_address=127.0.0.1:63318
loose-group_replication_group_seeds= "127.0.0.1:63318,127.0.0.1:53318,127.0.0.1:43318"
loose-group_replication_member_weight=50
loose-group_replication_flow_control_mode=disabled
slave_parallel_workers=256
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=on
```

需要注意的是，读者应修改 IP 地址以匹配其特定环境。

对于组复制，原生 MySQL 版本中从库的配置参数如下：

```
# for mgr
plugin_load_add='group_replication.so'
enforce-gtid-consistency
gtid-mode=on
loose-group_replication_member_expel_timeout=3
loose-group_replication_start_on_boot= OFF
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-baaaaaaaaaab"
loose-group_replication_local_address=127.0.0.1:53318
loose-group_replication_group_seeds= "127.0.0.1:63318,127.0.0.1:53318,127.0.0.1:43318"
loose-group_replication_member_weight=50
loose-group_replication_flow_control_mode=disabled
slave_parallel_workers=256
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=on
```

关于改进的组复制，由于 MySQL 8.0.32 和 MySQL 8.0.40 之间相似，我们在以下地址提供了一个可供在线使用的版本：https://github.com/enhancedformysql/mysql-8.0.40。

相应地，主库的配置参数如下：

```
# for mgr
plugin_load_add='group_replication.so'
enforce-gtid-consistency
gtid-mode=on
loose-group_replication_member_expel_timeout=3
loose-group_replication_start_on_boot= OFF
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-baaaaaaaaaab"
loose-group_replication_local_address=127.0.0.1:63318
loose-group_replication_group_seeds= "127.0.0.1:63318,127.0.0.1:53318,127.0.0.1:43318"
loose-group_replication_member_weight=50

slave_parallel_workers=256
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=on
```

对于改进的组复制，从库的配置参数如下：

```
# for mgr
plugin_load_add='group_replication.so'
enforce-gtid-consistency
gtid-mode=on
loose-group_replication_member_expel_timeout=3
loose-group_replication_start_on_boot= OFF
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-baaaaaaaaaab"
loose-group_replication_local_address=127.0.0.1:53318
loose-group_replication_group_seeds= "127.0.0.1:63318,127.0.0.1:53318,127.0.0.1:43318"
loose-group_replication_member_weight=50

slave_parallel_workers=256
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=on
```

请注意，我们不再提供基于 MySQL 8.0.32 的源代码，但我们提供了基于 MySQL 8.0.40 的源代码。

与半同步复制相关的详细信息可以在以下地址找到：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/semisynchronous.txt

### 7 源代码仓库

**"Percona Server for MySQL 8.0.27-18" 的补丁：**

补丁地址：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/book_8.0.27_single.patch

此补丁专门针对独立 MySQL 实例的优化，包括：

- **MVCC ReadView** 增强
- **Binlog 组提交** 改进
- **查询执行计划** 优化

**集群源代码：**

MySQL 集群版本的源代码可在此处获得：https://github.com/enhancedformysql/mysql-8.0.40

对于 MySQL 集群，补丁引入了对 **组复制** 和 **MySQL 从库回放** 的进一步优化。

## 关于作者

早年，王斌在一家专注于开发高性能计算和高并发系统的互联网公司工作。他还为 TCPCopy [65] 和 MySQL Proxy [66] 等开源项目贡献了力量，在问题解决方面获得了宝贵的经验，特别是在逻辑思维方面。

离开互联网公司后，他专注于 MySQL 相关的开发，成功为组复制、从库回放、InnoDB 存储引擎和查询优化 [67] 等项目贡献了力量。他在 MySQL 领域的问题解决方面积累了丰富的经验。
