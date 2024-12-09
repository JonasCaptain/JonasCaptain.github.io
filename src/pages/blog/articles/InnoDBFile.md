---
layout: "@layouts/ArticleLayout.astro"
title: MySQL文件
description: 主要摘抄至出版书籍
date: 2023-02-20 00:00:00
tags:
  - MySQL
  - InnoDB
  - File
  - Log
---


## 第3章 文件

### 1. 参数文件

> 默认情况下，MySQL实例会按照一定顺序在指定的位置进行读取配置文件，通过命令mysql --help|grep my.cnf 来寻找即可。

参数分为两种类型：

* 动态参数(dynamic)
* 静态参数(static)

动态参数在MySQL运行中可以进行修改，静态参数不能进行修改，类似于只读概念。

### 2. 日志文件

#### 2.1 错误日志(error log)

错误日志文件对MySQL的启动、运行、关闭过程进行了记录。遇到问题时应该首先查看该文件以便定位问题。该文件记录了所有的错误信息，也记录一些警告信息或正确的信息。用户可以通过命令show variables like 'log_error'来定位文件。

##### 2.2  慢查询日志(slow log)

MySQL启动时有一个参数long_query_time，默认值为10，代表10秒。当SQL执行时长超过此阈值时，将会记录SQL到慢查询日志中。因为执行时间过长的SQL往往都是有问题的SQL，需要优化。默认情况下，MySQL并不启动慢查询日志，需要手动将这个参数设置为ON。

```shell
mysql> show variables like 'long_query_time'; ## 查询阈值
mysql> show variables like 'log_slow_queries'; ## 查询慢日志开关
```

需要注意的是，运行时间刚好等于long_query_time的语句，并不会被记录下，也就是说，在源代码中判断的是大于，并非大于等于。从MySQL5.1开始，long_query_time开始以微秒记录SQL的运行时间。另一个和慢查询日志有关的参数是log_queries_not_using_indexes，如果运行的SQL语句没有使用索引，则MySQL数据库同样会将这条SQL语句记录到慢查询日志文件。首先确认打开了log_queries_not_using_indexes:

```shell
mysql> show variables like 'log_queries_not_using_indexes';
```

MySQL5.6.5版本开始新增了一个参数log_throttle_queries_not_using_indexes，用来表示每分钟允许记录到slow log的且未使用索引的SQL语句次数。该值默认为0，表示没有限制。在生产环境下，若没有使用索引，此类SQL语句会频繁被记录到slow log，从而导致slow log文件的大小不断增加。

随着MySQL运行时间的增加，可能会有越来越多的SQL被记录到慢查询日志中，此时要分析该文件就不那么简单直观了。可以使用数据库提供的mysqldumpslow命令，可以很好的解决该问题

```
mysqldumpslow nh122-190-slow.log
如果希望得到执行时间最长的10条SQL语句
mysqldumpslow -s al -n 10 david.log
```

MySQL5.1开始可以将慢查询的日志记录放入一张表中，慢查询表在mysql库下，名为slow_log，其表结构定义如下：

```shell
CREATE TABLE `slow_log` (
  `start_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `user_host` mediumtext NOT NULL,
  `query_time` time(6) NOT NULL,
  `lock_time` time(6) NOT NULL,
  `rows_sent` int NOT NULL,
  `rows_examined` int NOT NULL,
  `db` varchar(512) NOT NULL,
  `last_insert_id` int NOT NULL,
  `insert_id` int NOT NULL,
  `server_id` int unsigned NOT NULL,
  `sql_text` mediumblob NOT NULL,
  `thread_id` bigint unsigned NOT NULL
) ENGINE=CSV DEFAULT CHARSET=utf8 COMMENT='Slow log'
```

参数log_output指定了慢查询输出的格式，默认为FILE，可以将它设为TABLE，然后就可以查询mysql.slow_log表了。参数log_output是全局动态的，因此用户可以在线修改。查看slow_log表的定义会发现该表使用的引擎是CSV，对大数据量下的查询效率可能不高。用户可以把slow_log表的引擎切换到MyISAM，并在start_time列上添加索引进一步的提高查询效率。但是，如果已经启动了慢查询，将会提示错误。

MySQL的slow log通过运行时间来对SQL语句进行捕获，但是当数据库容量较小时，可能因为数据库刚建立，此时非常大的可能是数据全部缓存在缓冲池汇总，SQL语句运行的时间可能都是非常短的。

#### 2.3 查询日志(query log)

查询日志记录了所有对MySQL数据库请求的信息，无论这些请求是否得到了正确的执行。默认文件名：主机名.log。

#### 2.4 二进制日志(binary log)

二进制日志记录了对MySQL数据库执行更改的所有操作，但是不包括SELECT和SHOW这类操作，因为操作本身没有对数据进行修改。然而，若操作本身并没有导致数据库发生变化，那么该操作可能也会写入二进制日志。

二进制日志主要有以下几种作用：

* 恢复（recovery）：某些数据的恢复需要二进制日志，例如，在一个数据库全备文件恢复后，用户可以通过二进制日志进行point-in-time的恢复。
* 复制（replication）：其原理与恢复类似，通过复制和执行二进制日志使一条远程的MySQL数据库(一般称为slave或standby)与一台MySQL数据库(一般称为master或primary)进行实时同步。
* 审计（audit）：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入的攻击。

二进制日志文件在默认情况下并没有启动，需要手动指定参数来启动。开启二进制日志，会使性能下降1%，但考虑到可以使用复制（replication）和point-in-time的恢复，这些性能损失绝对是可以且应该被接受的。

以下配置文件的参数影响这二进制日志记录的信息和行为：

* max_binlog_size
* binlog_cache_size
* sync_binlog
* binlog-do-db
* binlog-ignore-db
* log-slave-update
* binlog_format

参数**max_binlog_size**指定了单个二进制日志文件的最大值，如果超过该值，则产生新的二进制文件，后缀名+1，并记录到.index文件。当使用事务的表存储引擎（如InnoDB）时，所有未提交（uncommitted）的二进制日志会被记录到一个缓存中去，等该事务提交（committed）时直接将缓冲中的二进制日志写入二进制日志文件，而该缓冲的大小有**binlog_cache_size**决定，默认大小为32K。此外binlog_cache_size是基于会话（session）的，也就是说，当月一个线程开始一个事务时，MySQL会自动分配一个大小为binlog_cache_size的缓存，因此该值的设置需要相当小心，不能设置过大。当一个事务的记录大于设定的binlog_cache_size时，MySQL会把缓冲中的日志写入一个临时文件中，因此该值又不能设置的太小。通过show global status命令查看binlog_cache_use、binlog_cache_disk_use的状态，可以判断当前binlog_cache_size的设置是否合适。binlog_cache_use记录了使用缓冲写二进制日志的次数，binlog_cache_disk_use记录了使用临时文件写二进制日志的次数。

```shell
mysql> show variables like 'binlog_cache_size';
+------------------+---------+
|Variable_name     |  Value  |
+------------------+---------+
|binlog_cache_size |  32768  |
+------------------+---------+
mysql> show global status like 'binlog_cache%';
+----------------------+---------+
|Variable_name         |  Value  |
+----------------------+---------+
|binlog_cache_disk_use |    0    |
|binlog_cache_use      |  33553  |
+----------------------+---------+
```

使用缓冲次数为33553，临时文件使用次数为0。

在默认情况下，二进制日志并不是在每次写的时候同步到磁盘（用户可以理解为缓冲写）。因此，当数据库所在操作系统发生宕机时，可能会有最后一部分数据没有写入二进制日志文件中，这会给恢复和复制带来问题。参数**sync_binlog**=[N]表示每写缓冲多少次就同步磁盘。如果将N设为1，即sync_binlog=1，表示采用同步写磁盘的方式来写二进制日志，这时写操作不使用操作系统的缓冲来写二进制日志。sync_binlog的默认值为0，如果使用InnoDB存储引擎进行复制，并且想得到最大的高可用性，建议将该值设为ON。不过该值为ON时，确实会对数据库的IO系统带来一定的影响。

参数**binlog-do-db**和**binlog-ignore-db**表示需要写入或忽略写入哪些库的日志。默认为空，表示需要同步所有库的日志到二进制日志。

如果当前数据库是复制中的slave角色，则它不会将从master取得并执行的二进制日志写入自己的二进制日志文件中去。如果需要写入，要设置**log-slave-update**。如果需要搭建master->slave->slave架构的复制，则必须设置该参数。

**binlog_format**参数十分重要，它影响了二进制日志的格式。该参数可设置的值有：STATEMENT、ROW、MIXED。

1. STATEMENT格式和之前的MySQL版本一样，二进制日志文件记录的是逻辑SQL语句。
2. 在ROW格式下，二进制日志记录的不再是简单的SQL语句，而是记录表的行更改情况。基于ROW格式的复制类似于Oracle的物理Standby（当然还是有区别）。同时，对上述提及的Statement格式下复制的问题予以解决。从MySQL5.1版本开始，如果设置了binlog_format为ROW，可以将InnoDB的事务隔离基本设为READ COMMITTED，以获得更好的并发性。
3. 在MIXED格式下，MySQL默认采用STATEMENT格式进行二进制日志文件的记录，但是在一些情况下会使用ROW格式，可能的情况有：
   1. 表的存储引擎为NDB，这是对表的DML操作都会以ROW格式记录。
   2. 使用了UUID()、USER()、CURRENT_USER()、FOUND_ROWS()、ROW_COUNT()等不确定函数。
   3. 使用了INSERT DELAY语句。
   4. 使用了用户定义的函数（UDF）
   5. 使用了临时表（temporary table）

此外，binlog_format参数还有对于存储引擎的限制

| 存储引擎      | Row格式 | Statement格式 |
| --------- | ----- | ----------- |
| InnoDB    | Yes   | Yes         |
| MyISAM    | Yes   | Yes         |
| HEAP      | Yes   | Yes         |
| MERGE     | Yes   | Yes         |
| NDB       | Yes   | No          |
| Archive   | Yes   | Yes         |
| CSV       | Yes   | Yes         |
| Federate  | Yes   | Yes         |
| Blockhole | No    | Yes         |

通常情况下，将参数binlog_format设置为ROW，可以为数据库的恢复和复制带来更好的可靠性。但是也会增加二进制文件的大小，有些语句下的ROW格式可能需要更大的容量。而且由于复制是采用传输二进制日志方式实现的，因此复制的网络开销也有所增加。

查看二进制日志文件的方式，通过mysqlbinlog命令完成。

#### 3. 套接字文件

在UNIX系统下本地连接MySQL可以采用UNIX域套接字方式，这种方式需要一个套接字(socket)文件。套接字文件可由参数socket控制。一般在/tmp/目录下，名为mysql.sock。

#### 4. pid文件

MySQL启动后，会将自己的进程ID写入一个文件中，这个文件就是pid文件。该文件可由参数pid_file控制，默认位于数据库目录下，文件名为主机名.pid。

#### 5. 表结构定义文件

因为MySQL插件式存储引擎体系结构的关系，MySQL数据的存储是根据表进行的，每个表都会有与之对应的文件。但不论表采用何种引擎，MySQL都有一个亿frm为后缀名的文件，这个文件记录了该表的表结构定义。frm还用来存放视图的定义，如用户创建了一个v_a视图，那么对应的会产生一个v_a.frm文件，用来记录视图的定义，该文件是文本文件，可以直接使用cat命令进行查看。

#### 6. InnoDB存储引擎文件

##### 6.1 表空间文件

InnoDB采用将存储的数据按表空间（tablespace）进行存放的设计。在默认配置下会有一个初始大小为10MB，名为ibdata1的文件。该文件就是默认的表空间文件（tablespace file），用户可以通过参数innodb_data_file_path对其进行设置。格式如下：

`innodb_data_file_path=datafile_spec1[;datafile_spec2]...`

用户可以通过多个文件组成一个表空间，同时制定文件的属性，如：

```ini
[mysqld]
innodb_data_file_path = /db/ibdata1:2000M;/dr2/db/ibdata2:2000M:autoextend
```

这里将/db/ibdata1和/dr2/db/ibdata2两个文件用来组成表空间。若这两个文件位于不同的磁盘上，磁盘的负载可能被平均，因此可以提高数据库的整体性能。同时，两个文件的文件名后都跟了属性，表示文件ibdata1的大小为2000MB，文件ibdata2的大小为2000MB，如果用完这2000MB，该文件可以自动增长(autoextend)。

设置innodb_data_file_path参数后，所有基于InnoDB存储引擎的表的数据都会记录到该共享表空间中。若设置了参数innodb_file_per_table，则用户可以将每个基于InnoDB存储引擎的表产生一个独立的表空间。独立表空间的命名规则为：表名.ibd。通过这样的方式，用户不用将所有的数据都存放与默认的表空间中。

需要注意的是，单独的表空间文件仅存储该表的数据、索引和插入缓冲BITMAP等信息，其余信息还是存放在默认的表空间中。

![Innodb表存储引擎体文件](/mysql/Innodb表存储引擎体文件.png)

######## 6.2 重做日志文件

在默认情况下，在InnoDB存储引擎的数据目录下会有两个名为ib_logfile0和ib_logfile1的文件。在MySQL官方手册中将其称为InnoDB存储引擎的日志文件，不过更准确的定义应该是重做日志文件（redo log file）。它们记录了对于INNODB存储引擎的事务日志。

当实例或介质失败（media failure）时，重做日志文件就能派上用场。例如，数据库由于所在主机掉电导致实例失败，InnoDB存储引擎会使用重做日志恢复到掉电前的时刻，以此来保证数据的完整性。

每个InnoDB存储引擎至少有1个重做日志文件组（group），每个文件组下至少有2个重做日志文件，如默认的ib_logfile0和ib_logfile1。为了得到更高的可靠性，用户可以设置多个的镜像日志组（mirrored log groups），将不同的文件组放在不同的磁盘上，以此提高重做日志的高可用性。在日志组中每个重做日志文件的大小一致，并以循环写入的方式运行。InnoDB存储引擎先写重做日志文件1，当达到文件的末尾时，会切换至重做日志文件2，再当重做日志文件2也被写满时，会切回到重做日志文件1中。

![InnoDB重做日志文件组](/mysql/InnoDB重做日志文件组.png)

下列参数影响这重做日志文件的属性：

* innodb_log_file_size
* innodb_log_files_in_group
* innodb_mirrored_log_groups
* innodb_log_group_home_dir

参数**innodb_log_file_size**指定每个重做日志文件的大小。

参数**innodb_log_files_in_group**指定了日志文件组中重做日志文件的数量，默认为2。

参数**innodb_mirrored_log_groups**指定了日志镜像文件组的数量，默认为1，表示只有一个日志文件组，没有镜像。若磁盘本身已经做了高可用的方案，如磁盘这列，那么可以不开启重做日志镜像的功能。

参数**innodb_log_group_home_dir**指定了日志文件组所在路径，默认为./，表示在MySQL数据库的数据目下。

------

重做日志文件的大小设置对于InnoDB存储引擎的性能有着非常大的影响。重做日志文件不能设置的太大，太大在恢复时可能需要很长的时间；太小，容易导致一个事务的日志需要多次切换重做日志文件。此外，重做日志文件太小会导致频繁发生async checkpoint，导致性能抖动。

既然同样是记录事务日志，与之前的二进制日志有什么区别？

首先，二进制日志会记录所有与MySQL数据库有关的日志记录，包括InnoDB、MyISAM、HEAP等其他存储引擎的日志。而InnoDB存储引擎的重做日志只记录有关自己本身的事务日志。

其次，记录的内容不同，无论用户将二进制日志文件记录的格式设为STATEMENT还是ROW，又或者是MIXED，其记录的都是关于一个事务的具体操作内容，即该日志是逻辑日志。而InnoDB存储引擎的重做日志文件记录的是关于每个页(page)的更改的物理情况。

此外，写入的时间也不同，二进制日志文件仅在事务提交前进行提交，即只写磁盘一次，不论这时该事务多大。而在事务进行的过程中，却不断有重做日志条目(redo entry)被写入到重做日志文件中。

------

写入重做日志文件的操作不是直接写，而是先写入一个重做日志缓冲(redo log buffer)中，然后按照一定的条件顺序地写入日志文件。

![重做日志写入过程](/mysql/重做日志写入过程.png)

从重做日志缓冲往磁盘写入时，是按512个字节，也就是一个扇区的大小进行写入。因为扇区是写入的最小单位，因此可以保证写入必定是成功的。因此在重做日志的写入过程中不需要有doublewrite。

**重做日志缓冲按照什么样的条件顺序写入日志文件？**

主线程(master thread)每秒会将重做日志缓冲写入磁盘的重做日志文件中，不论事务是否已经提交。另一个触发写磁盘的过程是有参数innodb_flush_log_at_trx_commit控制，表示在提交(commit)操作时，处理重做日志的方式。

参数**innodb_flush_log_at_trx_commit**的有效值为0、1、2。

* 0代表当事务提交时，并不将事务的重做日志写入磁盘上的日志文件，而是等待主线程每秒的刷新。
* 1表示在执行commit时将重做日志缓冲同步写到磁盘，即伴有fsync的调用。
* 2表示将重做日志异步写到磁盘，即写文件系统的缓存中。因此不能完全保证在执行commit时肯定会写入重做日志文件，只是有这个动作发生。

因此为了保证事务ACID中的持久性，必须将innodb_flush_log_at_trx_commit设置为1，也就是每当有事务提交时，就必须确保事务都已经写入重做日志文件。那么当数据库因为意外发生宕机时，可以通过重做日志文件恢复，并保证可以恢复已经提交的事务。而将重做日志文件设置为0或2，都有可能发生恢复时部分事务丢失。不同之处在于，设置为2时，当MySQL数据库发生宕机而操作系统及服务器并没有发生宕机时，由于此时未写入磁盘的事务日志保存在文件系统缓存中，当恢复时同样能保证数据不丢失。
