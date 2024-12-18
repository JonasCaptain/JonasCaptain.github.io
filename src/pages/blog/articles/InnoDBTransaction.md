---
layout: "@layouts/ArticleLayout.astro"
title: Innodb事务
description: 主要摘抄至出版书籍
date: 2023-02-20 00:00:00
tags:
  - MySQL
  - InnoDB
  - Transaction
  - 事务
---


## 第7章 事务

### 1. 事务的分类

从事务理论的角度来说，可以把事务分为以下几种类型：

* 扁平事务（Flat Transaction）
* 带有保存点的扁平事务（Flat Transaction with Savepoints）
* 链事务（Chained Transaction）
* 嵌套事务（Nested Transaction）
* 分布式事务（Distributed Transaction）

**扁平事务** 是事务类型中最简单的一种，但在实际生产环境中，这可能是使用最为频繁的事务。在扁平事务中，所有操作都处于同一层次，其由begin work开始，由commit work或rollback work结束，其间的操作都是原子的，因此 **扁平事务** 是应用程序成为原子操作的基本组成模块。

![](/mysql/InnoDB扁平事务的三种情况.png)

**扁平事务** 的主要限制是不能提交或者回滚事务的某一部分，或分几个步骤提交。这里给出一个扁平事务不足以支持的例子。例如用户在旅行网站上进行自己的旅行度假计划，用户设想从杭州到意大利的佛罗伦萨，这两个城市之间没有直达的班机，需要用户预订并转乘航班，或者需要搭火车等待。用户预订旅行度假的事务为：

```shell
Begin Work
## S1: 预订杭州到上海的高铁
## S2：上海浦东国际机场坐飞机，预订去米兰的航班
## S3：在米兰转火车前往佛罗伦萨，预订去佛罗伦萨的火车
```

但是当用户执行到S3时，发现由于飞机抵达米兰的时间太晚，已经没有当日的火车，于是决定在米兰住一晚，第二天出发。这时如果事务为扁平事务，则需要回滚S1、S2、S3的三个操作，这个代价就显得有点大。因为当再次进行该事务时，S1、S2的执行计划是不变的。也就是说，如果支持有计划的回滚操作，那么就不需要终止整个事务。因此就出现了 **带有保存点的扁平事务**。

**带有保存点的扁平事务**，除了支持扁平事务支持的操作外，允许在事务执行过程中回滚到同一事务中较早的一个状态。这是因为某些事务可能在执行过程中出现错误并不会导致所有的操作都无效，放弃整个事务不合乎要求，开销也太大。**保存点(Savepoint)** 用来通知系统应该记住事务当前的状态，以便当之后发生错误时，事务能回到保存点当时的状态。

对于扁平的事务来说，其隐式地设置了一个保存点。然而在整个事务中，只有这一个保存点，因此回滚只能回滚到事务开始时的状态。保存点用 save work 函数来建立，通知系统记录当前的处理状态。当出现问题时，保存点能用作内部的重启动点，根据应用逻辑，决定是回到最近一个保存点还是其他更早的保存点。

<img src="/mysql/InnoDB在事务中使用保存点.png" style="zoom:67%;" />

灰色部分的操作表示由rollback work而导致部分回滚，实际并没有执行的操作。当begin work开启一个事务时，隐式地包含了一个保存点，当事务通过rollback work：2发出部分回滚命令时，事务回滚到保存点2，接着一次执行，并再次执行到rollback work：7，直到最后commit work操作，这时表示事务结束，除灰色阴影部分的操作外，其余操作都已经执行，并且提交。

另一点需要注意的是，保存点在事务内部是递增的，从图中可以看出。有人可能会想，返回保存点2以后，下一个保存点可以为3，因为之前的工作都终止了。然而新的保存点编号为5，这意味着rollback 不影响保存点的计数，并且单调递增的编号能保持事务执行的整个历史过程，包括在执行过程中想法的改变。

此外，当事务执行rollback work：2命令发出部分回滚命令时，要记住事务并没有完全被回滚，只是回滚到了保存点2而已。这代表当前事务还是活跃的，如果想要完全回滚事务，还需要执行命令rollback work。

**链事务** 可视为保存点模式的一个变种。带有保存点的扁平事务，当 **发生系统崩溃** 时，所有的保存点都将消失，因为其保存点是易失的（volatile），而非持久的（persistent）。这意味着当进行恢复时，事务需要从开始处重新执行，而不能从最近的一个保存点继续执行。

链事务的思想是：在提交一个事务时，释放不需要的数据对象，将必要的处理上下文隐式地传给下一个要开始的事务。注意，提交事务操作和开始下一个事务操作将合并为一个原子操作。这意味着下一个事务将看到上一个事务的结果，就好像在一个事务中进行的一样。

![](/mysql/InnoDB链事务的工作方式.png)

链事务与带有保存点的扁平事务不同的是，带有保存点的扁平事务能回滚到任意正确的保存点。而链事务的回滚仅限于当前事务，即只能恢复到最近一个的保存点。对于锁的处理，两者也不相同。链事务在执行commit后即释放了当前事务所持有的锁，而带有保存点的扁平事务不影响迄今为止所持有的锁。

**嵌套事务** 是一个层次结构框架。由一个顶层事务（top-level transaction）控制着各个层次的事务。顶层事务之下嵌套的事务被称为子事务（subtransaction），其控制每一个局部的变换。

![](/mysql/InnoDB嵌套事务的结构层次.png)

下面给出Moss对嵌套事务的定义：

1. 嵌套事务是由若干事务组成的一颗树，子树既可以是嵌套事务，也可以是扁平事务。
2. 处在叶节点的事务是扁平事务。但是每个子事务从根到叶节点的距离可以是不同的。
3. 位于根节点的事务称为顶层事务，其他事务称为子事务。事务的前驱称（predecessor）为父事务（parent），事务的下一层称为儿子事务（child）。
4. 子事务既可以提交也可以回滚。但是它的提交操作并不马上生效，除非其父事务已经提交。因此可以推论出，任何子事务都在顶层事务提交后才真正的提交。
5. 树中任意一个事务的回滚会引起它的所有子事务一起回滚，故子事务仅保留A、C、I特性，不具有D的特性。

在Moss的理论中，实际的工作是交由叶子节点来完成的，即只有叶子节点的事务才能访问数据库、发送消息、获取其他类型的资源。而高层的事务仅负责逻辑控制，决定何时调用相关的子事务。即使一个系统不支持嵌套事务，用户也可以通过保存点技术来模拟嵌套事务。

**分布式事务** 通常是一个在分布式环境下运行的扁平事务，因此需要根据数据所在位置访问网络中的不同节点。

对于InnoDB存储引擎来说，其支持扁平事务、带有保存点的事务、链事务、分布式事务。对于嵌套事务，其并不原生支持，因此对有并行事务需求的用户来说，MySQL的InnoDB存储引擎就显得无能为力，不过还是可以通过带有保存点的事务来模拟。

### 2. 事务的实现

事务隔离性由锁来实现。原子性、一致性、持久性通过数据库的 **redo log** 和 **undo log** 来完成。redo log称为重做日志，用来保证事务的原子性和持久性。undo log称为回滚日志，用来保证事务的一致性。

redo和undo的作用都可以视为是一种恢复操作，redo恢复提交事务修改的页操作，而undo回滚行记录到某个特定版本。因此两者记录的内容不同，redo通常是物理日志，记录的是页的物理修改操作。undo是逻辑日志，根据每行记录进行记录。

------

######## 2.1 redo

########## 1. 基本概念

重做日志用来实现事务的持久性，即事务ACID中的D。其由两部分组成，一是内存中的日志缓冲（redo log buffer），其是易失的；而是重做日志文件（redo log file），其是持久的。

InnoDB是事务的存储引擎，其通过 **Force Log at Commit** 机制实现事务的持久性，即当事务提交（commit）时，必须先将事务的所有日志写入到重做日志文件进行持久化，待事务的commit操作完成才算完成。这里的日志是指重做日志，在InnoDB存储引擎中，由两部分组成，即redo log 和 undo log。redo log用来保证事物的持久性，undo log用来帮助事务回滚及MVCC的功能。redo log基本上都是顺序写的，在数据库运行时不需要对redo log的文件进行读取操作。而undo log是需要进行随机读写的。

为了保证

每次日志都写入重做日志，在每次将重做日志缓冲写入重做日志文件后，InnoDB存储引擎都需要调用一次fsync操作。由于重做日志文件打开并没有使用O_DIRECT选项，因此重做日志缓冲先写入文件系统缓存。为了确保重做日志写入磁盘，必须进行一次fsync操作。由于fsync的效率取决于磁盘的性能，因此磁盘的性能决定了事务提交的性能，也就是数据库的性能。

InnoDB存储引擎中的参数 **innodb_flush_log_at_trx_commit** 用来控制重做日志刷新到磁盘的策略。该参数的默认值为：1，表示事务提交时必须调用一次fsync操作。还可以设置该参数的值为0和2。0表示事务提交时不进行写入重做日志操作，这个操作仅在master thread中完成，而在master thread中每1秒会进行一次重做日志的fsync操作。2表示事务提交时将重做日志写入到文件系统的缓存中，不进行fsync操作，即由文件系统决定什么时间进行刷新磁盘。在2这个设置下，数据库宕机而操作系统不宕机，并不会导致事务丢失。若操作系统宕机，且文件系统并没有刷盘，重启数据库会丢失未刷盘的那部分事务。

有一个实验数据如下：

| innodb_flush_log_at_trx_commit | 执行所用时间     |
|:------------------------------:|:----------:|
| 0                              | 13.90s     |
| 1                              | 1min53.11s |
| 2                              | 23.37s     |

虽然用户可以通过修改参数为0或2来提高事务的性能，但是这个方式丧失了事务的ACID特性。

########## 2. log block

在InnoDB存储引擎中，重做日志都是以512字节进行存储的。这意味着重做日志缓存、重做日志文件都是以块（block）的方式进行保存的，称之为重做日志块（redo log block），每块的大小为512字节。

若一个页中产生的重做日志数量大于512字节 ，那么需要分割为多个重做日志块进行存储。此外，由于重做日志块的大小和磁盘扇区大小一样，都是512字节，因此重做日志的写入可以保证原子性，不需要doublewrite技术。

重做日志块除了日志本身之外，还由日志块头（log block header）及日志块尾（log block tailer）两部分组成。重做日志头一共占用12字节，重做日志尾占用8字节。故每个重做日志块实际可以存储的大小为492字节。

![](/mysql/InnoDB重做日志块缓存结构.png)

| 名        称                | 占用字节 |
|:-------------------------:|:----:|
| LOG_BLOCK_HDR_NO          | 4    |
| LOG_BLOCK_HDR_DATA_LEN    | 2    |
| LOG_BLOCK_FIRST_REC_GROUP | 2    |
| LOG_BLOCK_CHECKPOINT_NO   | 4    |

log buffer是有log block组成，在内部log buffer就好似一个数组，因此**LOG_BLOCK_HDR_NO**用来标记这个数组中的位置。其是递增并且循环使用的，占用4个字节，但是由于第一位用来判断是否是flush bit，所以最大值为2G。

**LOG_BLOCK_HDR_DATA_LEN**占用2字节，表示log block所占用的大小。当log block被写满时，该值为0x200，表示使用全部log block空间，即占用512字节。

**LOG_BLOCK_FIRST_REC_GROUP**占用2个字节，表示log back中第一个日志所在的偏移量。如果该值的大小和**LOG_BLOCK_HDR_DATA_LEN**相同，则表示当前log block不包含新的日志。如事务T1的重做日志1占用762字节，事务T2的重做日志占用100字节。由于每个log block实际只能保存492个字节，因此其在log buffer中的情况如图所示：

![](/mysql/log buffer示例图.png)

从图中观察得到，由于事务T1的重做日志占用762字节，因此需要占用两个log block。左侧的log block中的LOG_BLOCK_FIRST_REC_GROUP为12，即log block中第一个日志的开始位置（`自己注：因为log block header占用大小为12字节，且LOG_BLOCK_FIRST_REC_GROUP表示的是偏移量，真正的log body就是需要从头偏移12字节开始写入`）。在第二个log block中，由于包含了之前事务T1的重做日志，事务T2的日志才是log block的第一个日志，因此该log block的LOG_BLOCK_FIRST_REC_GROUP为282（270+12）。

**LOG_BLOCK_CHECKPOINT_NO**占用4个字节，表示该log block最后写入时的检查点第4个字节的值。

log block tailer只由1个部分组成，其值和LOG_BLOCK_HDR_NO相同，并在函数log_block_init中被初始化。

| 名      称         | 大小（字节） |
|:----------------:|:------:|
| LOG_BLOCK_TRL_NO | 4      |

########## 3. log group

log group为重做日志组，其中有多个重做日志文件。虽然源码中已支持log group的镜像功能，但是在ha_innobase.cc文件中禁止了该功能。因此InnoDB存储引擎实际上只有一个log group。

log group是一个逻辑上的概念，并没有一个实际存储的物理文件来表示log group信息。log group由多个重做日志文件组成，每个log group中的日志文件大小是相同的。

重做日志文件中存储的就是之前在log buffer中保存的log block，因此其也是根据块的方式进行物理存储的管理，每个块的大小与log block一样，同样为512字节。在InnoDB存储引擎运行过程中，log buffer根据一定的规则将内存中的log block刷新到磁盘。这个规则具体是：

* 事务提交时
* 当log buffer中有一半的内存空间已经被使用时
* log checkpoint时

对于log block的写入追加（append）在redo log file的最后部分，当一个redo log file 写满时，会接着写入下一个redo log file，其使用方式为round-robin。

虽然log block总是在redo log file的最后部分进行写入，但并不是顺序写入的。redo log file 除了保存log buffer刷新到磁盘的log block，还保存了一些其他的信息。对于log group中的第一个redo log file，其前2KB的部分保存4个512字节大小的块，其中存放的内容如下表：

| 名        称      | 大小（字节） |
|:---------------:|:------:|
| log file header | 512    |
| checkpoint1     | 512    |
| 空               | 512    |
| checkpoint2     | 512    |

需要特别注意的是，上述信息仅在每个log group的第一个redo log file中进行存储。log group中的其余redo log file仅保留这些空间，但不保存上述信息。正因为保存了这些信息，对redo log 的写入并不是完全顺序的。因为其除了log block的写入操作，还需要更新前2KB部分的信息，这些信息对于InnoDB存储引擎的恢复操作来说非常关键和重要。

在log file header后面的部分为InnoDB存储引擎保存的checkpoint值，其设计是交替写入，这样的设计避免了因介质失败而导致无法找到可用的checkpoint的情况。

########## 4. 重做日志格式

不同的数据库操作会有对应的重做日志格式。此外，由于InnoDB存储引擎的存储管理是基于页的，故其重做日志格式也是基于页的。虽然有着不同的重做日志格式，但是他们有着通用的头部格式，如图：

![](/mysql/InnoDB重做日志格式.png)

通用的头部格式由以下3部分组成：

* redo_log_type：重做日志的类型
* space：表空间的ID
* page_no：页的偏移量

之后的body部分，根据重做日志类型的不同，会有不同的存储内容，例如：

![](/mysql/InnoDB插入和删除的重做日志格式.png)

########## 5. LSN

LSN是Log Sequence Number的缩写，其代表的是日志序列号。在InnoDB存储引擎中，LSN占用8字节，并且单调递增。LSN表示的含义有：

* 重做日志写入的总量
* checkpoint的位置
* 页的版本

LSN不仅记录在重做日志，还存在于每个页中。在每个页的头部，有一个值FILE_PAGE_LSN，记录了该页的LSN。在页中，LSN表示该页最后刷新时LSN的大小。因为重做日志记录的是每个页的日志，因此页中的LSN用来判断是否需要进行恢复操作。

########## 6. 恢复

InnoDB存储引擎在启动时不管上次数据库运行是否正常关闭，都会尝试进行恢复操作。因为重做日志记录的是物理日志，因此恢复的速度比逻辑日志，如二进制日志，要快得多。与此同时，InnoDB存储引擎自身也对恢复进行了一定程度的优化，如顺序读取及并行应用重做日志，这样可以进一步地提高数据库恢复速度。

######## 2.2 undo

########## 1. 基本概念

重做日志记录了事务的行为，可以很好地通过其对页进行“重做”操作。但是事务有时还需要进行回滚操作，这是就需要undo。因此在对数据库进行修改时，InnoDB存储引擎不但会产生redo，还会产生一定量的undo。这样如果用户执行的事务或语句由于某些原因失败了，又或者用户用一条rollback语句请求回滚，就可以利用这些undo信息将数据回滚到修改之前的样子。

redo存放在重做日志文件中，与redo不同，undo存放在数据库内部的一个特殊段（segment）中，这个段称为undo段（undo segment）。undo段位于共享表空间内。undo是逻辑日志，只是将数据库逻辑地恢复到原来的样子。这是因为在多用户并发系统中，可能会有数十、数百甚至数千个并发事务。数据库的主要任务就是协调对数据记录的并发访问。比如，一个事务在修改当前一个页中某几条记录，同时还有别的事务在同一个页中另几条记录进行修改。因此，不能将一个页回滚到事务开始的样子，因为这样会影响其他事务正在进行的工作。

例如，用户执行了一个INSERT 10W条记录的事务，这个事务会导致分一个新的段，即表空间会增大。在用户执行rollback时，会将插入的事务进行回滚，但是表空间的大小并不会因此而收缩。因此，当InnoDB存储引擎回滚时，它实际上做的是与先前相反的操作。对于每个INSERT，InnoDB存储引擎会完成一个DELETE；对于每个DELETE，InnoDB存储引擎会执行一个INSERT；对于每个UPDATE，InnoDB存储引擎会执行一个相反的UPDATE，将修改前的行改回去。

除了回滚操作，undo的另一个作用就是MVCC，即在InnoDB存储引擎中MVCC的实现是通过undo来完成的。当用户读取一行记录时，若该记录已经被其他事务占用，当前事务可以通过unod读取之前的行版本信息，一次实现非锁定读取。

最后也是最为重要的一点，undo log会产生redo log，也就是undo log的产生会伴随着redo log的产生，这是因为undo log也需要持久性的保护。

########## 2. undo存储管理

InnoDB存储引擎对undo的管理同样采用段的方式。但是这个段和之前介绍的段有所不同。首先InnoDB存储引擎有rollback segment，每个回滚段中记录了1024个undo log segment，而在每个undo log segment段中进行undo页的申请。共享表空间偏移量为5的页(0,5)记录了所有rollback segment header所在的页，这个页的类型为FIL_PAGE_TYPE_SYS。

可通过参数对rollback segment进一步的设置。参数有：

* innodb_undo_directory
* innodb_undo_logs
* innodb_undo_tablespaces

参数 **innodb_undo_directory** 用于设置rollback segment文件所在的路径。意味着rollback segment可以存放在共享表空间以外的位置，即可以设置为独立表空间。该参数的默认值为“.”，表示当前InnoDB存储引擎的目录。

参数 **innodb_undo_logs** 用来设置rollback segment的个数，默认值为128。

参数 **innodb_undo_tablespaces** 用来设置构成rollback segment文件的数量，这样rollback segment可以较为平均的分布在多个文件中。设置该参数后，会在路径innodb_undo_directory看到undo为前缀的文件，该文件就代表rollback segment文件。

需要特别注意的是，事务在undo log segment分配页并写入undo log的这个过程同样需要写入重做日志。当事务提交时，InnoDB存储引擎会做以下两件事情：

* 将undo log放入列表中，供之后的purge操作
* 判断undo log所在的页是否可以重用，若可以分配给下个事务使用

事务提交后并不能马上删除undo log 及 undo log所在的页。这是因为可能还有其他的事务需要通过undo log 来得到行记录之前的版本。故事务提交时将undo log 放入一个链表中，是否可以最终删除 undo log及undo log所在页由purge线程来判断。

########## 3. undo log 格式

在InnoDB存储引擎中，undo log分为：

* insert undo log
* update undo log

**insert undo log** 是指insert操作中产生的undo log。因为insert操作的记录，只对事务本身可见，对其他事务不可见（这是事务隔离性的要求），故该undo log可以在事务提交后直接删除。不需要进行purge操作。

**update undo log** 比insert undo log 记录的内容更多，所需占用的空间也更大。next、start、undo_no、table_id与之前介绍的insert undo log 部分相同。

########## 4. 查看undo信息

######## 2.3 purge

delete和update操作可能并不直接删除原有的数据，把真正的删除操作放在了purge环节中完成。

purge用于最终完成delete和update操作。这样设计是因为InnoDB存储引擎支持MVCC，所以记录不能在事务提交时立即处理。这时其他事务可能正在引用，是否可以删除该条记录通过purge来判断。若该行记录已不被任何其他事务引用，那么就可以进行真正的delete操作。

######## 2.4 group commit

若事务为非只读事务，则每次事务提交时需要进行一次fsync操作，以此保证重做日志都已经写入磁盘。当数据库发生宕机时，可以通过重做日志进行恢复。虽然固态硬盘的出现提高了磁盘的性能，然而磁盘你的fsync性能是有限的。为了提高磁盘fsync的效率，当前数据库都提供了group commit的功能，即一次fsync可以刷新确保多个事务日志被写入文件。

###### 3. 事务控制语句

在MySQL命令行默认的设置下，事务都是自动提交的（auto commit）的，即执行SQL语句后就会马上执行commit操作。因此要显式的开启一个事务需要使用命令begin、start transaction，或者执行 set auto_commit=0，禁用当前会话的自动提交。

* start transaction|begin：显式的开启一个事务
* commit：提交事务，使得已对数据库做的所有修改成为永久性的。
* savepoint identifier：savepoint允许在事务中创建一个保存点，一个事务中可以有多个savepoint。
* release savepoint identifier：删除一个事务的保存点，当没有一个保存点执行这条语句时，会抛出一个异常。
* rollback to [savepoint] identifier：这个语句与savepoint命令一起使用。可以把事务回滚到标记点，而不会滚在此标记之前的任何工作。
* set transaction：这个语句用来设置事务的隔离级别。

commit和commit work都是用来提交事务。不同之处在于commit work用来控制事务结束后的行为是chain还是release。如果是chain，那么事务就变成了了链事务。

用户可以通过参数completion_type来进行控制，该参数默认为0，表示没有任何操作。此时commit 和 commit work是等价的。参数设置为1时，commit work 等同于commit and chain，表示马上自动开启一个相同隔离级别的事务。参数设置为2时，commit work等同于commit and release。在事务提交后会自动断开与服务器的连接。

###### 4. 隐式提交的SQL语句

以下这些SQL语句会产生一个隐式的提交操作，即执行完这些语句后，会有一个隐式的commit操作。

* DDL：alter database...upgrade data directory name, alter event, alter procedure, alter table ,alter view, create database, create event,create index, create procedure, create table, create trigger, create view, drop database, drop event, drop index, drop procedure, drop table, drop trigger, drop view, rename table, truncate table.
* 用来隐式的修改MySQL架构的操作：create user, drop user, grant, rename user, revoke, set password.
* 管理语句：analyze table, cache index, check table, load index into cache, optimize table, repair table.

###### 5. 对于事务操作的统计

由于InnoDB存储引擎是支持事务的，因此InnoDB存储引擎的应用需要在考虑请求数（Question Per Second， QPS）的同时，应该关注每秒事务处理的能力（Transaction Per Second，TPS）。

计算TPS的方法是（com_commit+com_rollback）/ time。但是利用这种方法进行的计算前提是：所有的事务都必须是显式提交的，如果存在隐式提交和回滚（默认autocommit=1），不会计算到com_commit和com_rollback变量中。

###### 6. 事务的隔离级别

SQL标准定义的4个隔离级别为：

* read uncommitted
* read committed
* repeatable read
* serializable

#### 第8章 备份与恢复

###### 1. 备份与恢复概述

可以根据不同的类型来划分备份的方法。根据备份的方法不同可以划分为：

* Hot Backup（热备）
* Cold Backup（冷备）
* Warm Backup（温备）

**Hot Backup** 是指数据库运行中直接备份，对正在运行的数据库操作没有任何的影响。这种方式在MySQL官方手册中称为 Online Backup（在线备份）。**Cold Backup** 是指备份操作在数据库停止的情况下，这种备份最简单，一般只需要复制相关的数据库物理文件即可。这种方式在MySQL官方手册中称为 Offline Backup（离线备份）。**Warm Backup** 备份同样是在数据库运行中进行的，但是会对当前数据库的操作有所影响，如加一个全局读锁以保证备份数据的一致性。

按照备份后文件的内容，备份又可以划分为：

* 逻辑备份
* 裸文件备份

在MySQL数据库中，**逻辑备份** 是指备份出的文件内容是可读的，一般是文本文件。内容一般是由一条条SQL语句，或者是表内实际数据组成。这类方法的好处是可以观察导出文件的内容，一般适用于数据库升级、迁移等工作。但其缺点是恢复需要的时间往往较长。

**裸文件备份** 是指复制数据库的物理文件，既可以是在数据库运行中的复制（如ibbackup、xtrabackup这类工具），也可以是在数据库停止运行时直接的数据文件复制。这类备份的恢复时间往往较逻辑备份短很多。

按照备份数据库的内容来分，备份可以划分为：

* 完全备份
* 增量备份
* 日志备份

**完全备份** 是指对数据库进行一个完整的备份。**增量备份** 是指上次完全备份的基础上，对于更改的数据进行备份。**日志备份** 主要是指对MySQL数据库二进制日志的备份，通过对一个完全备份进行二进制日志的重做（replay）来完成数据库的point-in-time的恢复工作。MySQL数据库复制（replication）的原理就是异步实时的将二进制日志重做传送并应用到从（slave/standby）数据库。

###### 2. 冷备

对于InnoDB存储引擎的冷备非常简单，只需要备份MySQL数据库的 frm 文件，共享空间文件，独立表空间文件（*.ibd），重做日志文件。另外建议定期备份MySQL数据库的配置文件my.cnf，这样有利于恢复的操作。

冷备的优点：

* 备份简单，只要复制相关文件即可。
* 备份文件易于在不同操作系统，不同MySQL版本上进行恢复。
* 恢复相当简单，只需要把文件恢复到指定位置即可。
* 恢复速度快，不需要执行任何SQL语句，也不需要重建索引。

冷备的缺点：

* InnoDB存储引擎冷备的文件通常比逻辑文件大很多，因为表空间中存放这很多其他的数据，如undo段，插入缓冲等信息。
* 冷备也不总是可以轻易的跨平台。操作系统、MySQL版本、文件大小写敏感和浮点数格式都会成为问题。

###### 3. 逻辑备份

######## 3.1 mysqldump

mysqldump的参数选项有很多，可以通过mysqldump --help命令来查看所有的参数，有些参数有缩写形式，如 --lock-tables的缩写形式 -l。这里举例一些比较重要的参数。

* --single-transaction：在备份开始前，先执行start transaction命令，以此来获得备份的一致性，当前该参数只对InnoDB存储引擎有效。当启用该参数并进行备份时，确保米有其他任何的DDL语句执行，因为一致性读并不能隔离DDL操作。
* --lock-tables（-l）：在备份中，以此锁住每个Schema下的所有表。一般用于MyISAM存储引擎，当备份时只能对数据库进行读取操作，不过备份依然可以保证一致性。对于InnoDB存储引擎，不需要使用该参数，用--single-transaction即可。并且--lock-tables和--single-transaction是互斥的，不能同时使用。如果既有MyISAM表又有InnoDB表，就能选择--lock-tables了，--lock-tables选项是依次对每个Schema中的表上锁的，因此只能保证每个Schema下表备份的一致性，而不能保证所有架构下表的一致性。
* --lock-all-tables（-x）：在备份过程中，对所有Schema中的所有表上锁。这个可以避免之前说的--lock-tables参数不能同时锁住所有表的问题。
* --add-drop-database：在create database前先运行drop database。这个参数需要和--all-databases或者--databases选项一起使用。在默认情况下，导出的文本文件中并不会有create database。
* --master-data [=value]：通过该参数产生的备份转存文件主要用来建立一个replication。当value的值为1时，转存文件中记录change master语句。当value的值为2时，change master语句被写出SQL注释。在默认情况下，value的值为空。
* --events（-E）：备份事件调度器。
* --routines（-R）：备份存储过程和函数。
* --triggers：备份触发器。
* --hex-blob：将binary、varbinary、blog和bit列类型备份为16进制的格式。mysqldump导出的文件一般都是文本文件，如果导出的类型中有前面几种，在文本模式下有些字符不可见，若添加--hex-blob选项，结果会以16进制的方式显示。
* --tab=path（-T path）：产生TAB分隔的数据文件。对于每张表，mysqldump创建一个包含create table语句的table_name.sql文件，和包含数据的tbl_name.txt文件。可以使用--fields-terminated-by=...，--fields-enclosed-by=...，--fields-optionally-enclosed-by=...，--fields-escaped-by=...，--lines-terminated-by=...来改变默认的分隔符、换行符。
* --where='where_condition'（-w 'where_conditon'）：导出给定条件的数据。

######## 3.2 select ... into outfile

select...into语句也是一种逻辑备份的方法，更准确的说是导出一张表的数据

######## 3.3 逻辑备份的恢复

mysqldump的恢复操作比较简单，因为导出的备份文件就是SQL语句，只需要执行这个文件就行了。

```shell
mysql -uroot -p < backup.sql
```

mysqldump不能导出视图，为了保证完全的恢复，在通过mysqldump导出存储过程、触发器、事件、数据后，还需要导出视图的定义或者备份视图定义的 frm 文件，并在恢复时进行导入。

######## 3.4 load data infile

若通过mysqldump-tab或者通过select into outfile导出的数据需要恢复，可以通过命令load data infile来进行导入。要对服务器文件使用load data infile，必须拥有FILE权限。

######## 3.5 mysqlimport

mysqlimport是MySQL数据库提供的一个命令行程序，本质上调用的是load data infile接口。

###### 4. 二进制日志备份与恢复

备份binlog，先通过flush logs生成一个新的binlog，然后再备份之前的binlog。

恢复binlog，通过mysqlbinlog

```shell
mysqlbinlog binlog.0000001 | mysql -uroot -p
```

###### 5. 热备

######## 5.1 ibbackup

ibbackup是InnoDB存储引擎官方提供的热备工具，可以同时备份MyISAM和InnoDB表。对于InnoDB表其备份的工作原理如下：

1. 记录备份开始时，InnoDB存储引擎重做日志文件检查点的LSN。
2. 复制共享表空间文件以及独立表空间文件。
3. 记录复制完表空间文件后，InnoDB重做日志文件检查点的LSN。
4. 复制在备份时产生的重做日志。

ibbackup优点：

* 在线备份，不阻塞任何的SQL语句。
* 备份性能好，备份的实质是复制数据库文件和重做日志文件。
* 支持压缩备份，通过选项，可以支持不同级别的压缩。
* 跨平台支持，ibbackup可以运行在Linux、Windows以及主流的UNIX系统平台上。

ibbackup对InnoDB的恢复步骤为：

* 恢复表空间文件
* 应用重做日志文件

######## 5.2 XtraBackup

XtraBackup备份工具是由Percona公司开发的开源热备工具。

######## 5.3 XtraBackup实现增量备份

Mysql数据库本身提供的工作并不支持真正的增量备份，准确的说，二进制日志恢复应该是point-in-time的恢复而不是增量备份。而XtraBackup工具支持对于InnoDB存储引擎的增量备份，其工作原理如下：

1. 首先完成一个全备，并记录下此时检查点的LSN。
2. 在进行增量备份时，比较表空间中每个页的LSN是否大于上次备份的LSN，如果是，则备份该页，同时记录当前检查点的LSN。

###### 6. 快照备份

Mysql数据库本身不支持快照功能，快照备份是基于文件系统支持的快照功能对数据进行备份。备份的前提是将所有数据库文件放在同一文件分区中，然后对该分区进行快照操作。

###### 7. 复制

######## 7.1 复制的工作原理

复制（replication）是MySQL数据库提供的一种高可用高性能的解决方案，一般用来建立大型的应用。replication的工作原理分为以下3个步骤：

1. 主服务器（master）把数据更改记录到二进制日志（binlog）中。
2. 从服务器（slave）把主服务器的二进制日志复制到自己的中继日志（relay log）中。
3. 从服务器重做中继日志中的日志，把更改应用到自己的数据库上，以达到数据的最终一致性。

复制的工作原理并不复杂，其实就是一个完全备份加上二进制日志备份还原。不同是这个二进制日志的还原操作基本上实时在进行中，这里特别需要注意的是，复制不是实时的同步，而是异步实时。这中间存在主从服务器之间的执行延时，如果主服务器压力很大，则可能导致主从延时较大。

![](/mysql/Mysql复制工作原理.png)

从服务器有2个线程，一个是I/O线程，负责读取主服务其的二进制日志，并将其保存为中继日志；另一个是SQL线程，复制执行中继日志。

show slave status的主要变量

| 变     量               | 说明                                                                                       |
| --------------------- | ---------------------------------------------------------------------------------------- |
| slave_io_state        | 显示当前IO线程的状态                                                                              |
| master_log_file       | 显示当前同步的主服务器的二进制日志                                                                        |
| read_master_log_pos   | 显示当前同步到主服务器上二进制日志的偏移量位置，单位是字节                                                            |
| relay_master_log_file | 当前中继日志同步的二进制日志                                                                           |
| relay_log_file        | 显示当前写入的中继日志                                                                              |
| relay_log_pos         | 显示当前执行到中继日志的偏移量位置                                                                        |
| slave_io_running      | 从服务器中IO线程的运行状态，YES表示正常                                                                   |
| slave_sql_running     | 从服务器中SQL线程的运行状态，YES表示正常                                                                  |
| exec_master_log_pos   | 表示同步到主服务器的二进制日志偏移量的位置。（Read_Master_Log_Pos - Exec_Master_Log_Pos）可以表示当前SQL线程运行的延时，单位是字节。 |

######## 7.2 快照+复制的备份架构

复制可以用来作为备份，但功能不仅限于备份，其主要功能如下：

* 数据分布。由于Mysql数据库提供的复制并不需要很大的带宽要求，因此可以在不同的数据中心之间实现数据的复制。
* 读取的负载平衡。通过建立多个从服务器，可将读取平均的分布到这些从服务器中，并且减少了主服务器的压力。一般通过DNS的Round-Robin和Linux的LVS功能都可以实现负载平衡。
* 数据库备份。复制对备份很有帮助，但是从服务器不是备份，不能完全替代备份。
* 高可用性和故障转移。通过复制建立的从服务器有助于故障转移，减少故障的停机时间和恢复时间。

可见，复制的设计不是简单用来备份的，并且只是用复制来进行备份是远远不够的。一个比较好的方法是通过对从服务器上的数据库所在的分区做快照，以此来避免误操作对复制造成的影响。当发生在主服务器上的误操作时，只需要将从服务器上的快照进行恢复，然后再根据二进制日志进行point-in-time的恢复即可。

![](/mysql/快照+备份的复制架构.png)

还有一些其他的方法来调整复制，比如采用延时复制，即间歇的开启从服务器上的同步，保证大约1小时的延时。这的确也是一个办法，只是数据库在高峰和非高峰期间每小时产生的二进制日志量是不同的，用户很难精确的控制。另外，这种方法也不能完全起到对误操作的防范作用。

此外，建议在从服务器上启用read-only选项，这样能保证从服务器上的数据仅与主服务器进行同步，避免其他线程修改数据。在启用read-only选项后，如果操作从服务器的用户没有super权限，则对从服务器进行任何的修改会抛出一个错误。

------
