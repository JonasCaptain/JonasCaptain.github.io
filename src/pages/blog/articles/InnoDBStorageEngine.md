---
layout: "@layouts/ArticleLayout.astro"
title: MySQL InnoDB存储引擎
description: 主要摘抄至出版书籍
date: 2023-03-20 10:00:00 +08:00
tags:
  - MySQL
  - InnoDB
  - StorageEngine
---
## 第2章

### 1. InnoDB体系架构

![Innodb存储引擎体系架构](/mysql/Innodb存储引擎体系架构.png)

从图中可见，InnoDB存储引擎有多个内存块，可以认为这些内存块组成了一个大的内存池，负责如下工作：

* 维护所有 进程 / 线程 需要访问的多个内部数据结构。
* 缓存磁盘上的数据，方便快速地读取，同时在对磁盘文件的数据修改之前在这里缓存。
* 重做日志(redo log)缓冲。
* ……

#### ***后台线程***

InnoDB存储引擎是多线程的模型，因此其后台有多个不同的后台线程，负责处理不同的任务。

1. **Master Thread**
   Master Thread是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲(INSERT BUFFER)、UNDO页的回收等。
2. **IO Thread**
   在InnoDB存储引擎中大量使用了AIO(Async IO)来处理写IO请求，这样可以极大提高数据库的性能。而IO Thread的工作主要是负责这些IO请求的回调(call back)处理。通过innodb_read_io_threads和innodb_write_io_threads参数进行设置。通过命令show engine innodb status来观察InnoDB中的IO Thread。
3. **Purge Thread**
   事务被提交后，其所使用的undo log可能不再需要，因此需要Purge Thread来回收已经使用并分配的undo页。可以在MySQL数据库的配置文件中添加如下命令来启用独立的Purge Thread：

   ```ini
   [mysqld]
   innodb_purge_threads=1
   ```

   在InnoDB 1.1版本中，即使将Innodb_purge_threads设为大于1，InnoDB存储引擎启动时也会将其设为1，并在错误文件中出现如下类似提示：

   ```
   120529 22:54:16 [Warning] option 'innodb-purge-threads': unsigned value 4 adjusted to 1
   ```

   从InnoDB 1.2版本开始，InnoDB支持多个Purge Thread，这样做的目的是为了进一步加快undo 页的回收。同时由于Purge Thread需要离散地读取undo 页，这样也能更进一步利用磁盘的随机读取性能。
4. **Page Cleaner Thread**

   Page Cleaner Thread是在InnoDB 1.2.X版本引入的。其作用是将之前版本中脏页的刷新操作都放入到单独的线程中来完成。而其目的是为了减轻原Master Thread的工作及对于用户查询线程的阻塞，进一步提高InnoDB存储引擎的性能。

#### ***内存***

1. **缓冲池**

   InnoDB存储引擎是基于磁盘处处的，并将其中的记录按照页的方式进行管理。因此可将其视为基于磁盘的数据库系统(Disk-base Database)。在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池技术来提高数据库的整体性能。

   大致原理如下：在数据库中进行读取页的操作，首先将从磁盘读到的页放在缓冲池中，这个过程称为将页“FIX”在缓冲池中。下一次读取相同页时，首先判断是否在缓冲池中。若在，称该页在缓冲池中被命中，直接读取该页。否则，读取磁盘上的页。对于数据库的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。这里需要注意的是，页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时触发，而是通过一种称为Checkpoint的机制刷新回磁盘。同样，这也是为了提高数据库的整体性能。

   对于InnoDB存储引擎而言，其缓冲池的配置通过参数innodb_buffer_pool_size来设置。

   具体来看，缓冲池中缓存的数据页类型有：索引页、数据页、undo 页、插入缓冲(insert buffer)、自适应哈希索引(adaptive hash index)、InnoDB存储的锁信息(lock info)、数据字典信息(data dictionary)等。不能简单的认为，缓冲池只是缓存索引页和数据页，它们只是占缓冲池很大的一部分而已。

   ![InnoDB内存数据对象](/mysql/InnoDB内存数据对象.png)

   从InnoDB 1.0.X版本开始，允许有多个缓冲池实例。每个页根据哈希值平均分配到不同的缓冲池实例中。这样做的好处是减少数据库内部的资源竞争，增加数据库的并发处理能力。可以通过参数innodb_buffer_pool_instances来进行配置，该值默认为1。在配置文件中奖innodb_buffer_pool_instances设置为大于1的值就可以得到多个缓冲池实例。再通过命令 show engine innodb status 可以观察到变化。

   从MySQL5.6版本开始，还可以通过information_schema架构下的表innodb_buffer_pool_stats来观察缓冲池的状态。
2. **LRU List、Free List 和 Flush List**

   通常来说，数据库中的缓冲池是通过LRU(Latest Recent Used，最近最少使用)算法来进行管理的。即最频繁使用的页在LRU列表的前端，而最少使用的页在LRU列表的尾端。当缓冲池不能存放新读取到的页时，将首先释放LRU列表中尾端的页。

   在InnoDB存储引擎中，缓冲池中页的大小默认为16KB，同样使用LRU算法对缓冲池进行管理。稍有不同的是InnoDB存储引擎对传统的LRU算法做了一些优化。在InnoDB存储引擎中，LRU列表中还加入了midpoint位置。新读取到的页，虽然是最新访问的页，但并不是直接放入到LRU列表的首部，而是放入LRU列表的midpoint位置。这个算法在InnoDB存储引擎下称为midpoint insertion strategy。在默认配置下，该位置在LRU列表长度的5/8处。midpoint位置可由参数innodb_old_blocks_pct控制。

   ```shell
   mysql>show variables like 'innodb_old_blocks_pct'\G
   ************************** 1. row **************************
   Variable_name: innodb_old_blocks_pct
           Value: 37
   1 row in set (0.00 sec)
   ```

   从上面的例子可以看到，参数innodb_old_blocks_pct默认值为37，表示新读取的页插入到LRU列表尾端的37%的位置(差不多3/8的位置)。在InnoDB存储引擎中，把midpoint之后的列表称为old列表，之前的列表称为new列表。可以简单的理解为new列表中的页都是最为活跃的热点数据。

   这样做的原因是，因为直接将读取到的页放入到LRU的首部，那么某些SQL操作可能会是缓冲池中的页被刷新出，从而影响缓冲池的效率。常见的这类操作为索引或数据的扫描操作。这类操作需要访问表中的许多页，甚至是全部的页，而这些页通常来说又仅在这次查询操作中需要，并不是活跃的热点数据。如果页被放入LRU列表的首部，那么非常可能将所需要的热点数据页从LRU列表中移除，而在下一次需要读取该页时，InnoDB存储引擎需要再次访问磁盘。为了解决这个问题，InnoDB存储引擎引入了另一个参数来以进一步管理LRU列表，这个参数是innodb_old_blocks_time，用于表示页读取到mid位置后需要等待多久才会被加入到LRU列表的热端。因此当需要执行上述所说的SQL操作时，可以通过下面的方法尽可能是LRU列表中的热点数据不被刷出。

   ```shell
   mysql> set global innodb_old_blocks_time=1000;
   Query OK, 0 rows affected (0.00 sec)

   ## data or index scan operation
   ## ...
   mysql> set global innodb_old_blocks_time=0;
   Query OK, 0 rows affected (0.00 sec)
   ```

   如果用户预估自己活跃的热点数据不止63%，那么在执行SQL语句前，还可以通过下面的语句来减少热点页不可能被刷出的概率。

   ```shell
   mysql> set global innodb_old_blocks_pct=20;
   Query OK, 0 rows affected (0.00 sec)
   ```

   LRU列表用来管理已经读取的页，但当数据库刚启动时，LRU列表是空的，即没有任何页。这时页都存放在Free列表中。当需要从缓冲池中分页时，首先从Free列表中查找是否有可用的空闲页，若有则将该页从Free列表中删除，放入到LRU列表中。否则，根据LRU算法，淘汰LRU列表末尾的页，将该内存空间分配给新的页。当页从LRU列表的old部分加入到new部分时，称此时发生的操作为 **page made young**，而因为innodb_old_blocks_time的设置而导致页没有从old部分移动到new部分的操作称为 **page not made young**。可以通过命令 show engine innodb status来观察LRU列表及Free列表的使用情况和运行状态。

   ```shell
   mysql> show engine innodb status \G
   =====================================
   2021-10-09 16:05:00 0x7feb9bb47700 INNODB MONITOR OUTPUT
   =====================================
   Per second averages calculated from the last 5 seconds
   -----------------
   BACKGROUND THREAD
   -----------------
   srv_master_thread loops: 1478477 srv_active, 0 srv_shutdown, 5524787 srv_idle
   srv_master_thread log flush and writes: 0
   ----------
   SEMAPHORES
   ----------
   OS WAIT ARRAY INFO: reservation count 4
   OS WAIT ARRAY INFO: signal count 4
   RW-shared spins 0, rounds 0, OS waits 0
   RW-excl spins 1, rounds 6, OS waits 0
   RW-sx spins 0, rounds 0, OS waits 0
   Spin rounds per wait: 0.00 RW-shared, 6.00 RW-excl, 0.00 RW-sx
   ------------
   TRANSACTIONS
   ------------
   Trx id counter 15483
   Purge done for trx's n:o < 15483 undo n:o < 0 state: running but idle
   History list length 0
   LIST OF TRANSACTIONS FOR EACH SESSION:
   ---TRANSACTION 422126106451512, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106450568, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106449624, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106447736, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106448680, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   ---TRANSACTION 422126106446792, not started
   0 lock struct(s), heap size 1136, 0 row lock(s)
   --------
   FILE I/O
   --------
   I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
   I/O thread 1 state: waiting for completed aio requests (log thread)
   I/O thread 2 state: waiting for completed aio requests (read thread)
   I/O thread 3 state: waiting for completed aio requests (read thread)
   I/O thread 4 state: waiting for completed aio requests (read thread)
   I/O thread 5 state: waiting for completed aio requests (read thread)
   I/O thread 6 state: waiting for completed aio requests (read thread)
   I/O thread 7 state: waiting for completed aio requests (read thread)
   I/O thread 8 state: waiting for completed aio requests (read thread)
   I/O thread 9 state: waiting for completed aio requests (read thread)
   I/O thread 10 state: waiting for completed aio requests (read thread)
   I/O thread 11 state: waiting for completed aio requests (read thread)
   I/O thread 12 state: waiting for completed aio requests (read thread)
   I/O thread 13 state: waiting for completed aio requests (read thread)
   I/O thread 14 state: waiting for completed aio requests (write thread)
   I/O thread 15 state: waiting for completed aio requests (write thread)
   I/O thread 16 state: waiting for completed aio requests (write thread)
   I/O thread 17 state: waiting for completed aio requests (write thread)
   I/O thread 18 state: waiting for completed aio requests (write thread)
   I/O thread 19 state: waiting for completed aio requests (write thread)
   I/O thread 20 state: waiting for completed aio requests (write thread)
   I/O thread 21 state: waiting for completed aio requests (write thread)
   I/O thread 22 state: waiting for completed aio requests (write thread)
   I/O thread 23 state: waiting for completed aio requests (write thread)
   I/O thread 24 state: waiting for completed aio requests (write thread)
   I/O thread 25 state: waiting for completed aio requests (write thread)
   Pending normal aio reads: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0] , aio writes: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0] ,
    ibuf aio reads:, log i/o's:, sync i/o's:
   Pending flushes (fsync) log: 0; buffer pool: 1
   874 OS file reads, 4458742 OS file writes, 100385 OS fsyncs
   0.00 reads/s, 0 avg bytes/read, 0.60 writes/s, 0.00 fsyncs/s
   -------------------------------------
   INSERT BUFFER AND ADAPTIVE HASH INDEX
   -------------------------------------
   Ibuf: size 1, free list len 0, seg size 2, 0 merges
   merged operations:
    insert 0, delete mark 0, delete 0
   discarded operations:
    insert 0, delete mark 0, delete 0
   Hash table size 138401, node heap has 0 buffer(s)
   Hash table size 138401, node heap has 0 buffer(s)
   Hash table size 138401, node heap has 0 buffer(s)
   Hash table size 138401, node heap has 0 buffer(s)
   Hash table size 138401, node heap has 1 buffer(s)
   Hash table size 138401, node heap has 1 buffer(s)
   Hash table size 138401, node heap has 2 buffer(s)
   Hash table size 138401, node heap has 4 buffer(s)
   0.00 hash searches/s, 0.00 non-hash searches/s
   ---
   LOG
   ---
   Log sequence number          18182045
   Log buffer assigned up to    18182045
   Log buffer completed up to   18182045
   Log written up to            18182045
   Log flushed up to            18182045
   Added dirty pages up to      18182045
   Pages flushed up to          18182045
   Last checkpoint at           18182045
   227 log i/o's done, 0.00 log i/o's/second
   ----------------------
   BUFFER POOL AND MEMORY
   ----------------------
   Total large memory allocated 549715968
   Dictionary memory allocated 463343
   Buffer pool size   32768 ## 有32768个页，即32768*16K，共512MB的缓冲池
   Free buffers       31720 ## 表示当前Free列表中页的数量
   Database pages     1040 ## 表示LRU列表中页的数量
   Old database pages 364
   Modified db pages  0
   Pending reads      0
   Pending writes: LRU 0, flush list 0, single page 0
   Pages made young 6049711, not young 12706
   0.00 youngs/s, 0.00 non-youngs/s
   Pages read 851, created 300576, written 4358280
   0.00 reads/s, 0.00 creates/s, 0.00 writes/s
   Buffer pool hit rate 1000 / 1000, young-making rate 71 / 1000 not 0 / 1000
   Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
   LRU len: 1040, unzip_LRU len: 0
   I/O sum[37]:cur[0], unzip sum[0]:cur[0]
   --------------
   ROW OPERATIONS
   --------------
   0 queries inside InnoDB, 0 queries in queue
   0 read views open inside InnoDB
   Process ID=543626, Main thread ID=140649955288832 , state=sleeping
   Number of rows inserted 67955720, updated 0, deleted 0, read 68608883
   9.60 inserts/s, 0.00 updates/s, 0.00 deletes/s, 9.60 reads/s
   Number of system rows inserted 47, updated 335, deleted 2, read 301015
   0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
   ----------------------------
   END OF INNODB MONITOR OUTPUT
   ============================
   ```

   可能的情况是Free buffers与Database pages的数量之和不等于Buffer pool size。因为缓冲池中的页还可能会被分配给自适应哈希索引、Lock信息、Insert Buffer等页，而这部分页不需要LRU算法进行维护，因此不存在与LRU列表中。

   **pages made young** 显示了LRU列表中页移动到前端的次数，因为设置了innodb_old_blocks_time的值，所以 **not young** 当前值为 12706。youngs/s、non-youngs/s表示每秒这两类操作的次数。这里还有一个重要的观察变量——Buffer pool hit rate，表示缓冲池的命中率。通常该值不应该小于95%，若发生Buffer pool hit rate小于95%这种情况，用户需要观察是否由于全表扫描引起的LRU列表被污染的问题。

   > Note：执行命令show engine innodb status显示的不是当前的状态，而是过去某个时间范围内InnoDB存储引擎的状态。上面截取的信息可以看出，Per second averages calculated from the last 5 seconds，代表的信息为过去5秒内的数据库状态。
   >

   从InnoDB 1.2版本开始，还可以通过表INNODB_BUFFER_POOL_STATS来观察缓冲池的运行状态

   ```shell
   mysql> select POOL_ID, HIT_RATE,
   -> PAGES_MADE_YOUNG, PAGES_NOT_MADE_YOUNG
   -> FROM information_schema.INNODB_BUFFER_POOL_STATS\G
   **************************** 1. row ****************************
                POOL_ID: 0
                  HIT_RATE: 980
       PAGES_MADE_YOUNG: 450
   PAGES_NOT_MADE_YOUNG: 0
   ```

   此外，还可以通过表INNODB_BUFFER_PAGE_LRU来观察每个LRU列表中每个页的具体信息，例如通过下面的语句可以看到缓冲池LRU列表中SPACE为1的表的页类型：

   ```shell
   mysql> select TABLE_NAME,SPACE,PAGE_NUMBER,PAGE_TYPE
       -> FROM INNODB_BUFFER_PAGE_LRU where SPACE = 1;
   +-------------------+-------+-------------+--------------------------+
   | TABLE_NAME        | SPACE | PAGE_NUMBER | PAGE_TYPE                |
   +-------------------+-------+-------------+--------------------------+
   | NULL              |     1 |           0 | FILE_SPACE_HEADER        |
   | NULL              |     1 |           1 | IBUF_BITMAP              |
   | NULL              |     1 |           2 | INODE                    |
   | test/t            |     1 |           3 | INDEX                    |
   +-------------------+-------+-------------+--------------------------+
   ```

   InnoDB存储引擎从1.0.X版本开始支持压缩页的功能，即将原本16KB的页压缩为1KB、2KB、4KB、8KB。由于页的大小发生了变化，LRU列表也有了些许改变。对于非16KB的页，是通过unzip_LRU列表进行管理的。通过命令show engine innodb status可以观察到如下内容：

   ```shell
   Buffer pool hit rate 1000 / 1000, young-making rate 71 / 1000 not 0 / 1000
   Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
   LRU len: 1539, unzip_LRU len: 156
   I/O sum[37]:cur[0], unzip sum[0]:cur[0]
   ```

   可以看到LRU列表中一共有1539个页，而unzip_LRU列表中有156个页。这里需要注意的是，LRU中的页包含了unzip_LRU列表中的页。

   对于压缩页的表，每个表的压缩比率可能各不相同。可能存在有的表页大小为8KB，有的表页大小为2KB的情况。

   **unzip_LRU是怎样从缓冲池中分配内存的呢？**

   首先，在unzip_LRU列表中对不同压缩页大小页进行分别管理。其次，通过伙伴算法进行内存分配。例如对需要从缓冲池汇总申请页为4KB的大小，其过程如下：

   1. 检查4KB的unzip_LRU列表，检查是否有可用的空闲页；
   2. 若有，则直接使用；
   3. 否则，检查8KB的unzip_LRU列表；
   4. 若能够得到空闲页，将页分成2个4KB页，存放到4KB的unzip_LRU列表；
   5. 若不能得到空闲页，从LRU列表中申请一个16KB的页，将页分为1个8KB的页，2个4KB的页，分别存放到对应的unzip_LRU列表中。

   同样可以通过information_schema.INNODB_BUFFER_PAGE_LRU来观察unzip_LRU列表中的页，如：

   ```shell
   mysql> select TABLE_NAME,SPACE,PAGE_NUMBER,COMPRESSED_SIZE
       -> from INNODB_BUFFER_PAGE_LRU
       -> where COMPRESSED_SIZE <> 0;
   +-------------------+-------+-------------+----------------------+
   | TABLE_NAME        | SPACE | PAGE_NUMBER | COMPRESSED_SIZE      |
   +-------------------+-------+-------------+----------------------+
   | sbtest/t          |     9 |         134 |                 8192 |
   | sbtest/t          |     9 |         135 |                 8192 |
   | sbtest/t          |     9 |          96 |                 8192 |
   | sbtest/t          |     9 |         136 |                 8192 |
   ....
   ```

   在LRU列表中的页被修改后，称该页未脏页(dirty page)，即缓冲池中的页和磁盘上的页的数据产生了不一致。这时数据库会通过checkpoint机制将脏页刷新回磁盘，而Flush列表中的页即为脏页列表。需要注意的是，脏页既存在与LRU列表中，也存在与Flush列表中。LRU列表用来管理缓冲池中页的可用性，Flush列表用来管理将页刷新回磁盘，二者互不影响。

   同LRU列表一样，Flush列表也可以通过show engine innodb status来查看，后面的数字表示了脏页的数量。

   ```shell
   Modified db pages  0
   ```

   information_schema库下并没有类似INNODB_BUFFER_PAGE_LRU表来显示脏页的数量及脏页的类型，但正如前面描述的那样，脏页同样存在与LRU列表中，故用户可以通过元数据表INNODB_BUFFER_PAGE_LRU来查看，唯一不同的是需要加入OLDEST_MODIFICATION大于0的SQL查询条件，如：

   ```shell
   mysql> select TABLE_NAME,SPACE,PAGE_NUMBER,PAGE_TYPE
       -> from INNODB_BUFFER_PAGE_LRU
       -> where OLDEST_MODIFICATION > 0;
   +-------------------+-------+-------------+-------------------+
   | TABLE_NAME        | SPACE | PAGE_NUMBER | PAGE_TYPE         |
   +-------------------+-------+-------------+-------------------+
   | NULL              |     0 |          56 | SYSTEM            |
   | NULL              |     0 |           0 | FILE_SPACE_HEADER |
   | test/t            |     1 |           3 | INDEX             |
   | NULL              |     0 |         320 | INODE             |
   | NULL              |     0 |         325 | UNDO_LOG          |
   +-------------------+-------+-------------+-------------------+
   ```

   可以看到当前共有5个脏页及它们对应的表和类型。TABLE_NAME为NULL表示该页属于系统表空间。
3. **重做日志缓冲**

   InnoDB存储引擎的内存区域除了有缓冲池外，还有重做日志缓冲(redo log buffer)。InnoDB存储引擎首先将重做日志信息先放入缓冲区，然后按一定频率将其刷新到重做日志文件。重做日志缓冲一般不需要设置的很大，因为一般情况下每一秒钟会将重做日志缓冲刷新到日志文件，因此用户只需要保证每秒产生的事务量在这个缓冲大小之内即可。该值可由配置参数innodb_log_buffer_size控制，默认为8MB。

   在通常情况下，8MB的重做日志缓冲池足以满足大部分的应用，因为重做日志在下列三种情况下会将重做日志缓冲中的内容刷新到外部磁盘的重做日志文件中。

   * Master Thread每一秒将重做日志缓冲刷新到重做日志文件；
   * 每个事务提交时会将重做日志缓冲刷新到重做日志文件；
   * 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件。
4. **额外的内存池**

   额外的内存池通常被DBA忽略，他们认为该值并不十分重要，事实恰恰相反，该值同样重要。在InnoDB存储引擎中，对内存的管理是通过一种称为内存堆(heap)的方式进行的。在对一些数据结构本身的内存进行分配时，需要从额外的内存池中进行申请，当该区域内存不够时，会从缓冲池中进行申请。例如，分配了缓冲池(innodb_buffer_pool)，但是每个缓冲池中的帧缓冲(frame buffer)还有对应的缓冲控制对象(buffer control block)，这些对象记录了一些诸如LRU、锁、等待等信息，而这个对象的内存需要从额外内存池中申请。因此，在申请了很大的InnoDB缓冲池时，也应考虑相应地增加这个值。

## 2. Checkpoint技术

**背景**：缓冲池解决了CPU与磁盘速度鸿沟的问题，但是每次更新都是先在缓冲池中完成，再刷新到磁盘，假如刷新过程突然宕机，数据就会丢失。

为了避免发生数据丢失问题，当前事务数据库系统普遍采用了**Write Ahead Log**策略，即当事务提交时，先写重做日志(redo log)，在修改页(即缓冲池中对应的数据)。当发生宕机时，通过重做日志来完成数据的恢复。这也是事务ACID中D(Durability持久性)的要求。

**思考**：如果重做日志可以无限地增大，同时缓冲池也足够大，能够缓冲所有数据库的数据，那么是不需要将缓冲池中页的新版本刷新回磁盘。因为发生宕机时，完全可以通过重做日志来恢复，但是这需要2个条件：

* 缓冲池可以缓存数据库中的所有数据；
* 重做日志可以无限增大。

对于第一个条件，刚建好的数据库还可以胜任，但是随着业务的扩展，用户的增加，缓冲池是不可能缓存下所有的数据，这是可以预见的。第二个条件，理论和实际都可以实现，但是成本要求太高，同时不便于运维。因为重做日志什么时候接近磁盘可使用空间的阈值是未知的，且还需要让存储设备支持动态扩展技术。

假如，2个条件都具备，那么还有一个情况需要考虑：宕机后数据库的恢复时间。当数据库运行了几个月甚至几年时，重新应用重做日志的时间会非常久，此时恢复的代价也会非常大。

**解决**：Checkpoint(检查点)技术的目的就是解决以下几个问题。

* 缩短数据库的恢复时间；
* 缓冲池不够用时，将脏页刷新到磁盘；
* 重做日志不可用时，刷新脏页。

当数据库发生宕机时，数据库不需要重做所有的日志，因为Checkpoint之前的页都已经刷新回磁盘。故数据库只需对checkpoint后的重做日志进行恢复。此外，当缓冲池不够用时，根据LRU算法会移除最近最少使用的页，若此页为脏页，那么需要强制执行Checkpoint，将脏页也就是页的新版本刷新回磁盘。重做日志出现不可用的情况是因为当前事务数据库对重做日志的设计都是循环使用的，并不是让其无限增大。重做日志可以被重用的部分都是不再需要的部分，当发生宕机时这部分日志就可以被覆盖重用。若此时重做日志还需要使用，那么必须强制产生Checkpoint，将缓冲池中的页至少刷新到当前重做日志的位置。对于InnoDB存储引擎而言，其是通过LSN(Log Sequence Number)来标记版本的。而LSN是8字节的数字，其单位是字节。每个页有LSN，重做日志也有LSN，Checkpoint也有LSN。可以通过show engine innodb status来观察：

```shell
---
LOG
---
Log sequence number          18182045
Log buffer assigned up to    18182045
Log buffer completed up to   18182045
Log written up to            18182045
Log flushed up to            18182045
Added dirty pages up to      18182045
Pages flushed up to          18182045
Last checkpoint at           18182045
227 log i/o's done, 0.00 log i/o's/second
```

在InnoDB存储引擎中，Checkpoint发生的时间、条件及脏页的选择等都非常复杂。而Checkpoint所做的事情无外乎是将缓冲池中的脏页刷回到磁盘。不同之处在于每次刷新多少页、每次从哪里取脏页，以及什么时间触发Checkpoint。在InnoDB存储引擎内部，有两种Checkpoint，分别为：

* Sharp Checkpoint
* Fuzzy Checkpoint

Sharp Checkpoint发生在数据库关闭时将所有的脏页都刷新回磁盘，这是默认的工作方式，即参数innodb_fast_shutdown=1。但是若数据库在运行时也使用Sharp Checkpoint，那么数据库的可用性就会受到很大影响。故在InnoDB存储引擎内部使用Fuzzy Checkpoint进行页的刷新，即只刷新一部分脏页，而不是刷新所有的脏页回磁盘。

在InnoDB存储引擎中可能发生如下几种情况的Fuzzy Checkpoint：

* Master Thread Checkpoint
* FLUSH_LRU_LIST Checkpoint
* Async/Sync Flush Checkpoint
* Dirty Page too much Checkpoint

Master Thread 发生的Checkpoint，差不多以每秒或每十秒的速度从缓冲池的脏页列表中的刷新一定比例的页回磁盘。这个过程是异步的，即此时InnoDB存储引擎可以进行其他的操作，用户查询线程不会阻塞。

FLUSH_LRU_LIST Checkpoint是因为InnoDB存储引擎需要保证LRU列表中需要有差不多100个空闲页可以使用。在InnoDB 1.1.X 版本之前，需要检查LRU列表中是否有足够的可用空间操作发生在用户查询线程中，显然这会阻塞用户的查询操作。倘若没有100个可用空闲页，那么InnoDB存储引擎会将LRU列表尾端的页移除。如果这些页中有脏页，那么需要进行Checkpoint，而这些页是来自LRU列表的，因此成为FLUSH_LRU_LIST Checkpoint。从MySQL5.6版本，也就是InnoDB 1.2.X开始，这个检查放在了一个单独的Page Cleaner线程中进行，并且用户可以通过参数innodb_lru_scan_depth控制LRU列表中可用页的数量，该值默认为1024，如：

```shell
mysql> show variables like 'innodb_lru_scan_depth'\G
**************************** 1. row ****************************
Variable_name: innodb_lru_scan_depth
        Value: 1024
```

Async/Sync Flush Checkpoint指的是重做日志不可用的情况，这时需要强制将一些页刷新回磁盘，而此时脏页是从脏页列表中选择的。若将已经写入到重做日志的LSN记为 redo_lsn，将已经刷新回磁盘最新页的LSN记为checkpoint_lsn，则可定义：

checkpint_age = redo_lsn - checkpoint_lsn

`定义checkpoint存在的时长,redo_lsn是最新一次写入redo log的lsn，checkpoint_lsn是最近一次刷新回磁盘的lsn，两者做差得到一个时间差值，即距离上一次刷新回磁盘经过了多长时间`

再定义以下变量：

async_water_mark = 75% * total_redo_log_file_size

sync_water_mark = 90% * total_redo_log_file_size

若每个重做日志文件的大小为1GB，并且定义了两个重做日志文件，则重做日志文件的总大小为2GB。那么async_water_mark = 1.5GB，sync_water_mark = 1.8GB。则：

* 当checkpoint_age < async_water_mark时，不需要刷新任何脏页到磁盘；
* 当async_water_mark<checkpoint_age<sync_water_mar时，触发Async Flush，从Flush列表中刷新足够的脏页回磁盘，使得刷新后满足checkpoint_age<async_water_mark；
* checkpoint_age > sync_water_mark，这种情况一般很少发生，除非设置的重做日志文件太小，并且在进行类似Load Data的Bulk Insert操作。此时触发Sync Flush操作，从Flush列表中刷新足够的脏页回磁盘，使得刷新后满足checkpoint_age<async_water_mark。

在InnoDB 1.2.X 版本之前，Async Flush Checkpoint会阻塞发现问题的用户的查询线程，而Sync Flush Checkpoint会阻塞所有的用户查询线程，并且等待脏页刷新完成。从InnoDB 1.2.X版本开始，也就是MySQL5.6版本，这部分的操作同样放入到了单独的Page Cleaner Thread中，故不会阻塞用户查询线程。

MySQL官方版本并不能查看刷新页是从Flush列表中还是从LRU列表中进行Checkpoint的，也不知道因为重做日志而产生的Async/Sync Flush次数。但是提供了方法，通过命令show engine innodb status来观察。（MySQL5.7版本没有找到相关数据）

最后一种Checkpoint的情况是Dirty Page too much，脏页太多，导致InnoDB存储引擎强制进行Checkpoint。其目的总得来说还是为了保证缓冲池中有足够可用的页。其由参数innodb_max_dirty_pages_pct控制：

```shell
mysql> show variables like 'innodb_max_dirty_pages_pct'\G
**************************** 1. row ****************************
Variable_name: innodb_max_dirty_pages_pct
        Value: 75
```

innodb_max_dirty_pages_pct值为75表示，当缓冲池中脏页的数量占据75%时，强制进行Checkpoint，刷新一部分的脏页到磁盘。

### 3. InnoDB关键特性

InnoDB存储引擎的关键特性包括：

* 插入缓冲（Insert Buffer）
* 两次写（Double Write）
* 自适应哈希索引（Adaptive Hash Index）
* 异步IO（Async IO）
* 刷新邻接页（Flush Neighbor Page）

1. Insert Buffer(插入缓冲)

在InnoDB存储引擎中，主键是行唯一的标识符。通常应用程序中行记录的插入顺序是按照主键递增的顺序进行插入的。因此，插入聚集索引（Primary Key）一般是顺序的，不许需要磁盘的随机读取。例如：表中字段设置了auto_increment属性，值就会自动增长，同时页中的行记录按该字段的值进行顺序存放。一般情况下，不许需要随机读取另一个页中的记录，因此对于这类情况下的插入操作，速度是非常快的。

> Note:
>
> 并不是所有的主键插入都是顺序的。若主键类是UUID这样的类，那么插入和辅助索引一样，同样是随机的。即使主键是自增类型，但是插入的是指定的值，而不是NULL值，那么同样可能导致插入并非连续的情况。

不过并不是每张表上只有一个聚集索引，更多情况下，一张表上有多个非聚集的辅助索引（Secondary index）。比如，用户需要按照b这个字段进行查找，并且b这个字段不是唯一的。

```shell
create table t (
    a int auto_increament,
    b varcahr(30),
    primary key(a),
    key(b)
)
```

在这样的情况下产生了一个非聚集索引且索引值不唯一。在进行插入操作时，数据页的存放还是按照主键a进行顺序存放，但是对于非聚集索引叶子节点的插入不再是顺序的了，这时就需要离散的访问非聚集索引页，由于随机读取的存在而导致了插入操作性能的下降。当然这并不是b字段上索引的错误，而是因为B+树的特性决定了非聚集索引插入的离散性。

InnoDB存储引擎设计了Insert Buffer，对于非聚集索引的插入或更新操作，不是每一次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入；若不在，则先放到一个Insert Buffer对象中。数据库这个非聚集的索引已经插到叶子节点，而实际并没有，只是存放在另一个位置。然后再以一定频率和情况进行Insert Buffer和辅助索引页子节点的merge（合并）操作，这时通常能将多个插入合并到一个操作中（因为在一个索引页中），这就大大提高了对于非聚集索引插入的性能。然而Insert Buffer的使用需要同时满足以下两个条件：

* 索引是辅助索引（secondary index）；
* 索引不是唯一（unique）的。

2. Change Buffer

InnoDB从1.0.X开始引入了Change Buffer，可将其视为Insert Buffer升级。从这个版本开始，InnoDB存储引擎可以对DML操作——Insert、Delete、Update都进行缓冲，他们分别是：Insert Buffer、Delete Buffer、Update Buffer。

和前面的Insert Buffer一样，Change Buffer使用的对象依然是非唯一的辅助索引。对一条记录进行update操作可能分为两个过程：

* 将记录标记为已删除；
* 真正将记录删除。

因此Delete Buffer对应update操作的第一个过程，即将记录标记为删除。Purge Buffer对应update操作的第二个过程，即将记录真正的删除。同时，InnoDB存储引擎提供了参数 innodb_change_buffering，用来开启各种Buffer的选项。该参数的可选值为：inserts、deletes、purges、changes、all、none。changes表示启用inserts和deletes，all表示启用所有，none表示都不启用。该参数默认值为all。

从1.2.X版本开始，可以通过参数 innodb_change_buffer_max_size 来控制Change Buffer最大使用内存的数量。

```shell
mysql> show VARIABLES like 'innodb_change_buffer_max_size'
**************************** 1. row ****************************
Variable_name: innodb_max_dirty_pages_pct
        Value: 25
```

innodb_change_buffer_max_size默认值为25，表示最多使用1/4的缓冲池内存空间。而需要注意的是，该参数的最大有效值为50。

```shell
## show engine innodb status
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
```

insert表示Insert Buffer；delete mark 表示Delete Buffer ；delete 表示 Purge Buffer；discarded operations 表示当Change Buffer发生merge时，表示已经删除，此时就无需再将记录合并（merge）到辅助索引中了。

### 3. Insert Buffer 的内部实现

Insert Buffer的数据结构是一颗B+树。在MySQL4.1之前的版本中每张表有一颗Insert Buffer B+ 树。而在现在的版本中，全局只有一颗Insert Buffer B+树，负责对所有的表的辅助索引进行Insert Buffer。而这颗B+树存放在共享表空间中，默认也就是ibdata1中。因此，试图通过独立表空间ibd文件恢复表中数据时，往往会导致CHECK TABLE 失败。这是因为表的辅助索引中的数据可能还在Insert Buffer中，也就是共享空间中，所以通过ibd文件进行恢复后，还需要进行REPAIR TABLE操作来重建表上所有的辅助索引。

### 4. Merge Insert Buffer

概括的说，Merge Insert Buffer 的操作可能发生在以下几种情况下：

* 辅助索引页被读取到缓冲池时；
* Insert Buffer Bitmap 页追踪到该辅助索引页已无可用空间时；
* Master Thread。

第一种情况为当辅助索引页被读取到缓冲池中时，例如这在执行正常的SELECT查询操作，这是需要检查Insert Buffer Bitmap页，然后确认该辅助索引页是否有记录存放于Insert Buffer B+ 树中。若有，则将Insert Buffer B+ 树中该页记录插入到该辅助索引页中。可以看到对该页多次的记录操作通过一次操作合并到了原有的辅助索引页中，因此性能会有大幅提高。

Insert Buffer Bitmap 页用来追踪每个辅助索引页的可用空间，并至少有1/32页的空间。若插入辅助索引记录时检测到插入记录后可用空间会小于1/32页，则会强制进行一个合并操作，即强制读取辅助索引页，将Insert Buffer B+数中该页的记录及待插入的记录插入到辅助索引中。这就是第二种情况。

在Master Thread线程中每秒或每10秒会进行一次 Merge Insert Buffer 操作，不同之处在于每次进行merge操作的页的数量不同。在Master Thread中，执行merge操作的不止是一个页，而是根据srv_innodb_io_capacity的百分比来决定真正要合并多少个辅助索引页。但InnoDB存储引擎又是根据怎样的算法来得知需要合并的辅助索引页呢？在Insert Buffer B+树中，辅助索引页根据（space，offset）都已排序好，故可以根据（space，offset）的排序进行页的选择。然而，对于Insert Buffer页的选择，InnoDB存储引擎并非采用这个方式，它随机地选择Insert Buffer B+树中的一个页，读取该页中的space及之后所需要数量的页。该算法在复杂情况下应有更好的公平性。同时，若进行merge时，要进行merge的表已经被删除，此时可以直接丢弃已经被Insert/Change Buffer的数据记录。

**两次写**

`如果说Insert Buffer带给InnoDB存储引擎是性能上的提升，那么double write带给InnoDB存储引擎的是数据页的可靠性。`

当发生数据库宕机时，可能InnoDB存储引擎正在写入某个页到表中，而这个页只写了一部分，比如16KB的页，只写了前4KB，之后就发生了宕机，这种情况被称为**部分写失效（partial page write）**。

![InnoDB_DoubleWrite架构](/mysql/InnoDB_DoubleWrite架构.png)

**Double Write**由两部分组成，一部分是内存中的doublewrite buffer，大小为2MB，另一部分是物理磁盘上共享表空间中连续的128个页，即2个区（extent），大小同样为2MB。在对缓冲池的脏页进行刷新时，并不直接写入磁盘，而是会通过memory函数将脏页先复制到内存中的doublewrite buffer，之后通过doublewrite buffer再分两次，每次1MB顺序地写入共享表空间的物理磁盘上，然后马上调用fsync函数，同步磁盘，避免缓冲写带来的问题。在这个过程中，因为doublewrite页是连续的，因此这个过程是顺序写的，开销并不是很大。在完成doublewrite页的写入后，再将doublewrite buffer中的页写入哥哥表空间文件中此时的写入则是离散的。可以通过命令show global status like 'innodb_dblwr%'来观察。

```shell
mysql> show global status like 'innodb_dblwr%';
**************************** 1. row ****************************
Variable_name: innodb_dblwr_pages_written
        Value: 6325194
**************************** 2. row ****************************
Variable_name: innodb_dblwr_writes
        Value: 100399
```

doublewrite一共写了6325194个页，但实际写入次数为100399，基本上符合64:1。如果发现系统在高峰时的Innodb_dblwr_pages_written:Innodb_dblwr_writes远小于64:1，那么可以说明系统写入压力并不是很高。

如果操作系统在将页写入磁盘的过程中发生了崩溃，在恢复过程中，InnoDB存储引擎可以从共享表空间中的doublewrite中找到该页的一个副本，将其复制到表空间文件，再应用重做日志。

若查看MySQL官方手册，会发现在命令show global status中Innodb_buffer_pool_pages_flushed变量表示当前从缓冲池中刷新磁盘页的数量。根据之前的介绍，在默认情况下所有页的刷新首先都需要放入到doublewrite中，因此该变量应该和Innodb_dblwr_pages_written一致。

参数skip_innodb_doublewrite可以禁止使用doublewrite功能，这时可能会发生前面提及的写失效问题。不过如果用户有多个从服务器(slave server)，需要提供较快的性能(如在slave server上做的是RAID 0)，也许启用这个参数是一个办法。不过对于需要提高数据高可靠性的主服务器(master server)，任何时候都应确保开启doublewrite功能。

> Note:
>
> 有些文件系统本身就提供了部分写失效的防范机制，如ZFS文件系统。在这种情况下，用户就不要启用doublewrite了。

#### 3. 自适应哈希索引

`哈希(hash)是一种非常快的查找方法，一般情况下的时间复杂度为O(1)。而B+树的查找次数，取决于B+树的高度，生产环境中，B+树的高度一般为3~4层，故需要3~4次的查询。`

InnoDB存储引擎会监控对表上各索引页的查询。如果观察到建立哈希索引可以带来速度提升，则建立哈希索引，称之为自适应哈希索引（Adaptive Hash Index，AHI）。AHI是通过缓冲池的B+树页构造而来，因此建立的速度很快，而且不需要对整张表构建哈希索引。InnoDB存储引擎会自动根据访问频率和模式来自动地为某些热点页建立哈希索引。

AHI有一个要求，即对这个页的连续访问模式必须是一样的。例如对于（a，b）这样的联合索引页，其访问模式可以是以下情况：

* WHERE a=XXX
* WHERE a=XXX and b=XXX

访问模式一样指的是查询条件一昂，若交替执行上述两种查询，那么InnoDB存储引擎不会对该页构造AHI。此外还有如下要求：

* 以该模式访问了100次
* 页通过该模式访问了N次，其中N=页中记录 * 1/16

根据InnoDB存储引擎官方文档显示，启用AHI后，读取和写入速度可以提高2倍，辅助索引的连续操作性能可以提高5倍。AHI是非常好的优化模式，其设计思想是数据库自优化(self-tuning)，即无需DBA对数据库进行人为调整。通过命令show engine innodb status观察使用情况。

```shell
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 138401, node heap has 0 buffer(s)
Hash table size 138401, node heap has 0 buffer(s)
Hash table size 138401, node heap has 0 buffer(s)
Hash table size 138401, node heap has 0 buffer(s)
Hash table size 138401, node heap has 1 buffer(s)
Hash table size 138401, node heap has 1 buffer(s)
Hash table size 138401, node heap has 2 buffer(s)
Hash table size 138401, node heap has 4 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
```

哈希索引只能用来搜索等值的查询，如where inde_col = 'xxx'。而对于其他类型的查找，如范围查找，是不能使用哈希索引的，因此这里出现了non-hash searchs/s的情况。通过 hash searchs:non-hash searchs可以大概了解使用哈希索引后的效率。

由于AHI是有InnoDB存储引擎控制的，因此这里的信息仅供参考。不给过可以通过观察show engine innodb status的结果及参数 innodb_adaptive_hash_index来考虑是禁用或启用此特性，默认AHI为开启状态。

#### 4. 异步IO

为了提高磁盘操作性能，当前的数据库系统都采用异步IO(Asynchronous IO，AIO)的方式来处理磁盘操作。AIO的一个优势是可以进行IO Merge操作，也就是将多个IO合并为1个IO，这样可以提高IOPS的性能。

在InnoDB 1.1.X之前，AIO的实现通过InnoDB存储引擎中的代码来模拟实现。而从1.1.X开始，提供了内核级别的AIO支持，称为Native AIO。因此在编译或者运行该版本的MySQL时，需要libaio库的支持。如没有则会出现如下提示：

```shell
/usr/local/mysql/bin/mysqld: error while loading shared libraries: libaio.so.1: cannot open shared object file: No such file or directory
```

需要注意的是，Native AIO 需要操作系统提供支持。Windows和Linux都支持，而Mac OSX则未提供。参数innodb_use_native_aio用来控制是否启用Native AIO，在Linux操作系统下，默认值为ON。

#### 5. 刷新邻接页

工作原理：当刷新一个脏页时，InnoDB存储引擎会检测该页所在去（extent）的所有页，如果是脏页，那么一起进行刷新。这样做的好处显而易见，通过AIO可以将多个IO写入操作合并为一个IO操作，故该工作机制在传统机械磁盘下有着显著的优势。但是需要考虑下面两个问题：

* 是不是可能将不怎么脏的页进行了写入，而该页之后又会很快变成脏页？
* 固态硬盘有着较高的IOPS，是否还需要这个特性？

为此，InnoDB存储引擎从1.2.X开始提供了参数innodb_flush_neighbors，用来控制是否启用该特性。对于传统机械硬盘建议启用该特性，而对于固态硬盘有着超高IOPS性能的磁盘，则建议将该参数设置为0。
