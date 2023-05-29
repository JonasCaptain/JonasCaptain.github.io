---
layout: "@layouts/ArticleLayout.astro"
title: MySQL 表
description: 主要摘抄至出版书籍
date: 2023-02-20 00:00:00
tags:
  - MySQL
  - Table
  - Index
---

## 第4章 表

### 1. 索引组织表

在InnoDB存储引擎中，表都是根据主键顺序组织存放的，这种存储方式的表称为索引组织表(index organized table)。在InnoDB存储引擎表中，每张表都有个主键(primary key)，如果在创建表时没有显式地定义主键，则InnoDB存储引擎会按如下方式选择或创建主键：

* 首先判断表汇总是否有非空的唯一索引(Unique Not Null)，如果有，则改了即为主键。
* 如果不符合上述条件，InnoDB存储引擎自动创建一个6字节大小的指针。

当表中有多个非空唯一索引时，InnoDB存储引擎将选择建表时第一个定义的非空唯一索引为主键。需要注意的是，主键的选择根据的是定义索引的顺序，而不是建表时列的顺序。

### 2. InnoDB逻辑存储结构

从InnoDB存储引擎的逻辑存储结构看，所有数据都被逻辑地存放在一个空间中，称为表空间(tablespace)。表空间又由段(segmetn)、区(extent)、页(page)组成。页在一些文档中有时也称为块(block)，InnoDB存储引擎的逻辑存储结构大致如图。

![](/mysql/InnoDB存储引擎逻辑存储结构.png)

#### 2.1 表空间（Tablespace）

表空间可以看做是InnoDB存储引擎逻辑结构的最高层，所有的数据都放在表空间中。默认情况下，有一个共享表空间ibdata1，即所有数据都存放在这个表空间内。如果用户启用了参数innodb_file_per_table，则每张表内的数据可以单独放到一个表空间。

如果启用了innodb_file_per_table的参数，需要注意的是每张表的表空间内存放的只是数据、索引和插入缓冲bitmap页，其他类的数据，如回滚(undo)信息、插入缓冲索引页、系统事务信息、二次写缓冲(Double write buffer)等还是存放在原来的共享表空间内。即使启用了参数innodb_file_per_table之后，共享表空间还是会不断地增加其大小。

#### 2.2 段（Segment）

表空间是由段组成的，常见的段有数据段、索引段、回滚段等。因为InnoDB存储引擎表是索引组织的，因此数据即索引，索引即数据。那么数据段即为B+树的叶子节点（Leaf node segment），索引段即为B+树的非索引节点（Non-Leaf node segment）。

#### 2.3 区（extent）

区是由连续页组成的空间，在任何情况下每个区的大小都为1MB。为了保证区中页的连续性，InnoDB存储引擎一次从磁盘申请4~5个区。在默认情况下，InnoDB存储引擎页的大小为16KB，即一个区中一共有64个连续的页。InnoDB 1.0.X开始引入压缩页，每个页的大小可以通过参数key_block_size设置为2K、4K、8K，因此每个区对应页的数量应该为512、256、128。

InnoDB 1.2.X版本新增了参数innodb_page_size，通过该参数可以将默认页的大小设置为4K、8K，但是页中的数据库不是压缩。 这是区中页的数量同样也为256、128。总之，无论页的大小如何变化，区的大小总是1MB。

#### 2.4 页（page）

同大多数数据库一样，InnoDB有页的概念，页是InnoDB磁盘管理的最小单位。默认每个页大小为16KB，通过参数也可以设置为4K、8K、16K。

在InnoDB存储引擎中，常见的页类型：

* 数据页（B-tree Node）
* undo页（undo Log Page）
* 系统页（System Page）
* 事务数据页（Transaction system Page）
* 插入缓冲位图页（Insert Buffer Bitmap）
* 插入缓冲空闲列表页（Insert Buffer Free List）
* 未压缩的二进制大对象页（Uncompressed BLOB Page）
* 压缩的二进制大对象页（Compressed BLOB Page）

#### 2.5 行（row）

InnoDB存储引擎是面向列的(row-oriented)，也就是数据是按行进行存放的。每个页存放的行记录也是有硬性定义的，最多允许存放16KB/2-200行的记录，即7992行记录。既然有row-oriented，也就有column-oriented数据库。MySQL infobright存储引擎就是按列来存放数据的。

### 3. InnoDB行记录格式（略过）

### 4. InnoDB数据页结构（略过）

### 5. 约束

#### 5.1 数据完整性

一般来说，数据完整性有以下三种形式：

* 实体完整性保证表中有一个主键。
  
  在InnoDB存储引擎表中，用户可以通过定义Primary Key或Unique Key约束来保证实体的完整性。用户还可以通过编写一个触发器来保证数据的完整性。

* 域完整性保证数据每列的值满足特定的条件。
  
  在InnoDB存储引擎表中，域完整性可以通过以下几种途径来保证：
  
  * 选择合适的数据类型确保一个数据值满足特定的条件。
  * 外键约束（Foreign Key）。
  * 还可以考虑用default约束作为强制域完整性的一个方面。

* 参照完整性保证两张表之间的关系。
  
  InnoDB存储引擎支持外键，因此允许用户定义外键以强制参照完整性，也可以通过编写触发以强制执行。

对于InnoDB存储引擎本身而言，提供了以下几种约束：

* Primary Key
* Unique Key
* Foreign Key
* Default
* NOT NULL

######## 5.2 约束的创建和查找

约束创建可以采用以下两种方式：

* 表建立时就进行约束定义
* 利用ALTER TABLE命令来进行创建约束

#### 5.3 约束和索引的区别

约束是一个逻辑的概念，用来保证数据的完整性，而索引是一个数据结构，既有逻辑上的概念，在数据库中还代表着物理存储的方式。

#### 5.4 对错误数据的约束

在某些默认设置下，MySQL数据库允许非法的或不正确的数据的插入或更新，又或者可以在数据库内部将其转化为一个合法的值，如向NOT NULL的字段插入一个NULL值，MySQL数据库会将其更改为0再进行插入，因此数据库本身没有对数据的正确性进行约束。比如往date列插入了一个非法的日期'2009-02-30'。MySQL并没有报错，而是显示了警告(Waring)。如果用户想通过约束对数据库非法数据的插入或更更新，那么用户必须设置参数sql_mode，用来严格审核输入的参数。

#### 5.5 ENUM和SET约束

#### 5.6 触发器与约束

#### 5.7 外键约束

外键用来保证参照完整性，MySQL下的MyISAM存储引擎本身不支持外键，对于外键的定义只是起到一个注释作用。外键的定义如下：

```mysql
[CONSTRAINT] [symbol] FOREIGN KEY
[index_name] (index_col_name, ...)
REFERENCES tbl_name (index_col_name, ...)
[ON DELETE reference_option]
[ON UPDATE reference_option]
reference_option:
RESTRICT | CASCADE | SET NULL | NO ACTION
```

可以在CREATE TABLE时就添加外键，也可以在表创建后通过ALTER TABLE命令来添加。一般来说，被引用的表为父表，引用的表称为子表。外键定义时的ON DELETE和ON UPDATE表示在对父表进行DELETE和UPDATE操作时，对子表所做的操作，可以定义的子表操作有：

* CASCADE
* SET NULL
* NO ACTION
* RESTRICT

CASCADE表示当父表发生DELETE或UPDATE操作时，对相应的子表中的数据页进行DELETE或UPDATE操作。

SET NULL表示父表发生DELETE或UPDATE操作时，相应子表中的数据被更新为NULL值，但是子表中的列必须允许为NULL值。

NO ACTION表示父表发生DELETE或UPDATE操作时，抛出错误，不允许这类操作发生。

RESTRICT表示父表发生DELETE或UPDATE操作时，抛出错误，不允许这类操作发生。如果定义外键没有指定ON DELETE或ON UPDATE，RESTRICT就是默认的外键设置。

#### 6. 视图

在MySQL数据库中，视图（View）是一个命名的虚表，它由一个SQL查询来定义，可以当做表使用。与持久表（permanent table）不同的是，视图中的数据没有实际的物理存储。

##### 6.1 视图的作用

视图的主要用途之一是被用做一个抽象装置，特别是对于一些应用程序，程序本身不需要关心基表（base table）的结构，只需要按照视图定义来取数据或更新数据，因此视图同时在一定程度上起到一个安全层的作用。

```mysql
CREATE [OR REPLACE]
[ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
[DEFINER = {user | CURRENT_USER}]
[SQL SECURITY {DEFINER | INVOKER}]
VIEW view_name [(column_list)]
AS select_statement
[WITH [CASCADE | LOCAL] CHECK OPTION]
```

虽然视图是基于基表的一个虚拟表，但是用户可以对某些视图进行更新操作，其本质就是通过视图的定义来更新基本表。一般称可以进行更新操作的视图为可更新视图（updateable view）。视图定义中的 with check option就是针对于可更新的视图的，即更新的值是否需要检查。

##### 6.2 物化视图

Oracle支持物化视图——该视图不是基于基表的虚表，而是根据基表实际存在的实表，即物化视图的数据存储在非易失的存储设备上。物化视图可以用于预先计算并保存多表的链接（JOIN）或聚集（GROUP BY）等耗时较多的SQL操作结果。这样，在执行复杂查询时，就可以避免进行这些耗时的操作，从而快速得到结果。物化视图的好处是对于一些复杂的统计类查询能直接查出结果。

在Oracle中，物化是的创建方式包括以下两种：

* BUILD IMMEDIATE
* BUILD DEFERRED

BUILD IMMEDIDATE是默认的创建方式，在创建物化视图的时候就生成数据，而BUILD DEFERRED则在创建物化视图时不生成数据，以后根据需要再生成数据。查询重写是指当对物化视图的基表进行查询时，数据库会自动判断能否通过查询物化视图来直接得到最终的结果，如果可以，则避免了聚集或连接等这类较为复杂的SQL操作，直接从已经计算好的物化视图中得到所需的数据。物化视图的刷新是指当基表发生了DML操作，物化视图何时采用哪种方式和基表进行同步。刷新的的模式有两种：

* ON DEMAND
* ON COMMIT

ON DEMAND意味着物化视图在用户需要的时候进行刷新，ON COMMIT意味着物化视图在对基表的DML操作提交时的同事进行刷新。而刷新的方式有4种：

* FAST
* COMPLETE
* FORCE
* NEVER

FAST采用增量刷新，只刷新自上次刷新以后进行的修改。COMPLETE是对整个物化视图进行完全的刷新。FORCE，数据库在刷新时会去判断是否可以进行快速刷新，如果可以，则用FAST，否则采用COMPLETE。NEVER是物化视图不进行任何刷新。

MySQL本身并不支持物化视图，换句话说，MySQL数据库中的视图总是虚拟的。但是用户可以通过一些机制来实现物化视图的功能。例如要创建一个 ON DEMAND的物化视图还是比较简单的，用户只需定时把数据导入到另一张表。

### 7. 分区表

#### 7.1 分区概述

分区功能并不是在存储引擎层完成的，因此不是只有InnoDb才支持分区，常见的存储引擎MyISAM、NDB都支持。CSV、FEDORATED、MERGE等就不支持。

分区的过程是将一个表或索引分解为多个更小、更可管理的部分。就访问数据库的应用而言，从逻辑上讲，只要一个表或一个索引，但是在物理上这个表或索引可能由数十个物理分区组成。每个分区都是独立的对象，可以独自处理，也可以作为一个更大的对象的一部分进行处理。

MySQL数据库支持的分区类型为水平分区(将同一表中不同**行的记录**分配到不同的物理文件中)，并不支持垂直分区(将同一表中不同**列的记录**分配到不同物理文件中)。此外，MySQL数据库的分区是局部分区索引，一个分区中既存放了数据又存放了索引。而全局分区是指，数据存放在各个分区中，但是所有数据的索引放在一个对象中。可以通过以下命令查看当前数据库是否启用了分区功能：

```mysql
mysql> show variables like '%partition%';
Variable_name: have_partitioning
        Value: YES
```

当前MySQL数据库支持以下几种类型的分区：

* RANGE分区：行数基于属于一个给定连续区间的列值被放入分区
* LIST分区：和RANGE分区，只是LIST分区面向的是离散的值
* HASH分区：根据用户自定义的表达式的返回值来进行分区，返回值不能为负数
* KEY分区：根据MySQL数据库提供的哈希函数来进行分区

无论创建何种类型的分区，如果表中存在主键或唯一索引时，分区列必须是唯一索引的一个组成部分。

```mysql
create table t1(
    col1 int not null,
    col2 date not null,
    col3 int not null,
    col4 int not null,
    unique key(col1,col2)
)
partition by hash(col3)
partitions 4;
ERROR 1503: A PRIMARY KEY must include all columns in the table's partitioning function
```

唯一索引可以是允许NULL值的，并且分区列只要是唯一索引的一个组成部分，不需要整个唯一索引都是分区列。

```mysql
create table t1(
    col1 int null,
    col2 date null,
    col3 int null,
    col4 int null,
    unique key (col1, col2, col3, col4)
)
partition by hash(col3)
partitions 4;
Query OK, 0 rows affected
```

如果建表时没有指定主键，唯一索引，可以指定任何一个列作为分区列，因此下面两句SQL都是正确的。

```mysql
create table t1 (
    col1 int null,
    col2 date null,
    col3 int null,
    col4 int null
) engine=innodb
partition by hash(col3)
partitions 4;

create table t1 (
    col1 int null,
    col2 date null,
    col3 int null,
    col4 int null,
    key (col4)
) engine=innodb
partition by hash(col3)
partitions 4;
```

#### 7.2 分区类型(略)

------
