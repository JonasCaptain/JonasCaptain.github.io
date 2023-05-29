---
layout: "@layouts/ArticleLayout.astro"
title: MySQL锁
description: 主要摘抄至出版书籍
date: 2023-03-20T00:00:00Z
tags:
  - MySQL
  - Lock
  - 锁
---

## 数据库锁

### 1.MySQL都有什么锁，死锁判定原理和具体场景，死锁怎么解决？

#### MySQL有三种锁

* 表级锁：开销小，加锁快；不会出现死锁；锁粒度最大，发生锁冲突的概率最大，并发度最低
* 行级锁：开销大，加锁慢；会出现死锁；锁粒度最小，发生冲突的概率最低，并发度也最高
* 页级锁：开销和加锁的时间介于表级锁和行级锁；会出现死锁；锁粒度介于表锁和行锁之间，并发度一般

#### 什么是死锁？

**死锁：**指两个或两个以上的进程在执行过程中，因争夺资源而造成的一项互相等待的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。

表级锁不会产生死锁，所以解决死锁主要还是针对于最常用的InnoDB。

**死锁的关键在于：**两个（或以上）Session加锁的顺序不一致。那么对应的解决死锁问题的关键就是，让不同的session加锁有次序。

#### 死锁的解决办法

1. 查出死锁的线程并kill掉

   ```mysql
   SELECT trx_MySQL_thread_id FROM information_schema.INNODB_TRX;
   ```

2. 设置锁的超时时间
   
   InnoDB行锁的等待时间，单位秒，可在会话级别设置，RDS实例该参数的默认值为50秒
   
   生产环境不推荐使用过大的 inoodb_lock_wait_timeout 参数值，该参数支持在会话级别修改，方便应用在会话级别单独设置某些特殊操作的行锁等待超时时间。如下：
   
   ```mysql
   set innodb_lock_wait_timeout = 1000;
   ```

3. 指定获取锁的顺序

#### 2. 有哪些锁，SELECT时怎么加排它锁？

##### 加锁机制

* 悲观锁（Pessimistic Lock）：
  
  特点：先获取锁，在进行业务操作。
  
  即“悲观”的认为获取锁是非常有可能失败的，因此要先确保获取锁成功在进行业务操作。通常所说的“一锁二查三更新”即指的是使用悲观锁。通常来讲在数据库上的悲观锁需要数据库本身提供自持，即通过常用的 select .... for update 操作来实现悲观锁。当数据库执行select .... for update时会获取被select 中的数据行的行锁，因此其他并发执行的select .... for update如果试图选择同一行则会发生排斥(需要等待行锁被释放)，因此达到锁的效果。select .... for update 获取的行锁会在当前事务结束时自动释放，因此必须在事务中使用。
  
  **补充**：不同的数据库对 select .... for update 的实现和支持都是有区别的。
  
  * Oracle支持 select .... for update no wait，表示如果拿不到锁立刻报错，而不是等待，MySQL就没有 no wait这个选项。
  * MySQL还有个问题是 select .... for update语句执行中所有扫描过的行都会被锁上，这一点很容易造成问题。因此如果在MySQL中用悲观锁务必要确定走了索引，而不是全表扫描。

* 乐观锁（Optimistic Lock）：
  
  1. 乐观锁，也叫乐观并发控制，它假设多用户并发的事务在处理时不会彼此互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。在提交数据更新之前，每个事务会先检查在该事务读取数据库后，有没有其他事务又修改了该数据。如果其他事务有更新的话，那么当前正在提交的事务会进行回滚。
  
  2. 乐观锁的特点是先进行业务操作，不得万不得已不去拿锁。即“乐观”的认为拿锁多半是会成功的，因此在进行完业务操作需要实际更新的最后一步再去拿锁就好。乐观锁在数据上的实现完全是逻辑的，不需要数据库提供特殊的支持。
  
  3. 一般的做法是在需要锁的数据上增加一个版本号，或者时间戳。
     
     实现方式举例如下：
     
     乐观锁（给表加一个版本号字段），这个并不是乐观锁的定义，给表加版本号，是数据实现乐观锁的一种方式。
     
     1. seelct data as old_data, version as old_version from ...;
     2. 根据获取的数据进行业务操作，得到new_data和new_version
     3. update table set data = new_data, version = new_version where version = old_version
     
     ```code
     if (updated row > 0 ) {
     
     ​    //乐观锁获取成功，操作完成
     
     } else {
     
     ​    //乐观锁获取失败，回滚并重试
     
     }
     ```
     
     **注意：**
     
     * 乐观锁在不发生取锁失败的情况下开销比乐观锁小，但是一旦发生失败回滚开销则比较大，因此适合用在取锁失败概率比较小的场景，可以提升系统并发性能
     * 乐观锁还适用于一些特殊的场景，例如在业务操作过程中无法和数据库保持连接等悲观锁无法使用的地方

#### **总结：**

悲观锁和乐观锁是数据库用来保证数据并发安全防止更新丢失的两种方法。悲观锁和乐观锁大部分场景下差异不大，一些独特的场景下有一些差异，一般可以从如下几个方面来判断：

* 响应速度：如果需要非常高的响应速度，建议采用乐观锁方案，成功就执行，不成功就失败，不需要等待其他并发去释放锁。
* 冲突频率：如果冲突频率非常公安，建议是采用悲观锁，保证成功率，如果冲突频率大，乐观锁会需要多次重试才能成功，代价比较大。
* 重试代价：如果重试代价大，建议采用悲观锁。

#### 扩展知识点

行级锁是MySQL中锁定粒度最细的一种锁，行级锁能大大减少数据库操作的冲突，行级锁分为共享锁和排它锁两种。

**共享锁(Share Lock)：**

共享锁又称读锁，是读取操作创建的锁。其他用户可以并发读取共享锁，不能加排它锁。获取共享锁的事务只能读取数据，不能修改数据。

用法：

```sql
select ... lock in share mode;
```

在查询语句后面增加lock in share mode，MySQL就会对查询结果中的每行都加共享锁，当没有其他线程对查询结果集中的任何一行使用排它锁时，可以成功申请共享锁，否则会被阻塞。其他线程也可以读取使用了共享锁的表，而且这些线程读取的是同一个版本的数据。

**排它锁(Exclusive Lock)：**

排它锁又称写锁、独占锁，如果事务T对数据A加上排它锁后，则其他事务不能再对A加任何类型的封锁。获取排它锁的事务既能读取数据，又能修改数据。

用法：

```sql
select ... for update;
```

在查询语句后面加上 for update，MySQL就会对查询结果集中的每一行都加排它锁，当没有其他线程对查询结果集中的任何一行使用排它锁时，可以成功申请排它锁，否则会被阻塞。

**意向锁(Intention Lock)：**

意向锁是表级锁，其设计目的主要是为了在一个事务中揭示下一行将要被请求锁的类型。

InnoDB中的两个表锁：

* 意向共享锁(IS)：表示事务准备给数据行加入共享锁，也就说一个数据行加共享锁前必须先取得该表的IS锁；
* 意向排它锁(IX)：类似上面，表示事务准备给数据行加入排它锁，说明事务在一个数据行加排它锁前必须先取得该表的IX锁；

意向锁是InnoDB自动加的，不需要用户干预。

对于INSERT、UPDATE和DELETE，InnoDB会自动给涉及的数据加排它锁；对于一般的SELECT语句，InnoDB不会加任何锁，事务可以通过以下语句显式加共享锁或排它锁。

```sql
## 共享锁
select ... lock in share mode;
## 排它锁
select ... for update;
```

#### 意向锁的引入解决了什么问题？

> 假设，事务A获取了某一行的排它锁，尚未提交，此时事务B想要获取表锁时，必须要确认表的每一行都不存在排它锁，很明显效率会很低，引入意向锁之后，效率就会大为改善。

1. 如果事务A获取了某一行的排它锁，实际此表存在两种锁，表中某一行的排它锁和表上的意向排它锁；
2. 如果事务B试图在该表级别上加锁时，则受到上一个意向锁的阻塞，它在锁定该表前不必检查各个页或行锁，而只需要检查表上的意向锁。

**宏观角度理解锁(图)**
![mysql锁](/mysql/MySQL-锁.jpg)

从锁粒度角度来讲，InnoDB允许行级锁与表级锁共存，而意向锁是表锁；从锁模式角度看，意向锁是一种独立类型，辅助解决记录锁效率不及的问题；从兼容性角度，意向锁包含了共享/排它两种。

**总结下意向锁的几个特征：**

1. 意向锁是表锁。做兼容性比照时，一定要分清是表锁还是行锁，如下图所示：
   
   | 兼容性    | IS  | IX  | S   | X   |
   | ------ | --- | --- | --- | --- |
   | **IS** | 兼容  | 兼容  | 兼容  | 不兼容 |
   | **IX** | 兼容  | 兼容  | 不兼容 | 不兼容 |
   | **S**  | 兼容  | 不兼容 | 兼容  | 不兼容 |
   | **X**  | 不兼容 | 不兼容 | 不兼容 | 不兼容 |
   
   这里的排它（X）、共享（S）,说的是表锁。表级锁不会和行级锁做比对，这里特别容易混淆。意向锁不会与行级锁中的共享(S)、排它(X)互斥。

2. 用户无法操作意向锁，意向锁是由InnoDB自己维护的。说白了，意向锁是帮助InnoDB提高效率的一种手段。

举例：

表

| ID  | name      |
| --- | --------- |
| 1   | ROADHOG   |
| 2   | Reinhardt |
| 3   | Tracer    |
| 4   | Genji     |
| 5   | Hanzo     |
| 6   | Mccree    |

事务A先获取了某一行的排它锁，并未提交：

```sql
select * from users where id = 6 for update;
```

1. 事务A获取了users表上的**意向排它锁**。
2. 事务A获取了id为6的数据行上的**排它锁**。

之后事务B想要获取users表的共享锁：

```sql
## 这是共享锁，表级锁
lock tables users READ;
## 这是排它锁，表级锁
lock tables user WRITE;
```

1. 事务B检测到事务A持有users表的**意向排它锁**。
2. 事务B对users表的加锁请求被阻塞（排斥）。

最后事务C也想获取users表中某一行的排它锁：

```sql
select * from users where id = 5 for update;
```

1. 事务C申请users表的**意向排它锁**。
2. 事务C检测到事务A持有users表的**意向排它锁**。
3. 因为意向锁之间并不互斥，所以事务C获取到了users表的意向排它锁。
4. 因为id为5的数据行上不存在任何**排它锁**，最终事务C成功获取到了该数据行上的排它锁。

**总结：**

1. InnoDB支持多粒度锁，特定场景下，行级锁可以与表级锁共存。
2. 意向锁之间互不排斥，但除了IS与S兼容外，意向锁会与 共享 / 排它锁互斥。
3. IX，IS是表级锁，不会和行级的X，S发生冲突，只会和表级的X、S发生冲突。
4. 意向锁在保证并发行的前提下，实现了行锁和表锁共存且满足事务隔离性的要求。

#### 3.行锁的实现方式（Record Lock）

InnoDB行锁是通过给索引上的**`索引项`**（*通常是主键列或者唯一索引*）加锁来实现的，否则加的锁就会变成`Next-Key Lock`。同时查询语句必须为精准匹配（=），不能为 > 、 < 、 like等，否则也会退化成`Next-Key Lock`。这一点MySQL与Oracle不同，候着是通过在数据块中相对应数据行加锁来实现的。InnoDB这种行锁实现特点意味着：只要通过索引条件检索数据，InnoDB才会使用行级锁，否则InnoDB将使用表级锁。

> 实际应用中，需要特别注意InnoDB行锁这一特性，不然的话，可能会导致大量的锁冲突，从而影响性能。

1. 在不通过索引条件查询的时候，InnoDB确实使用的是表锁，而不是行锁。

在如下所示的例子中，开始tab_no_index表没有索引：

```sql
create table tab_no_index(id int,name varchar(10)) engine=innodb;  
## Query OK, 0 rows affected (0.15 sec)  
insert into tab_no_index values(1,'1'),(2,'2'),(3,'3'),(4,'4');  
## Query OK, 4 rows affected (0.00 sec)  
## Records: 4  Duplicates: 0  Warnings: 0  
```

| session_1                                                                                                                                                                                                                                              | session_2                                                                                                                                                                                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)<br/>mysql> select * from tab_no_index where id = 1 ;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br>1 row in set (0.00 sec) | mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)<br/>mysql> select * from tab_no_index where id = 2 ;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 2  \| 2  \|<br/>+------+------+<br/>1 row in set (0.00 sec) |
| mysql> select * from tab_no_index where id = 1 for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br/>1 row in set (0.00 sec)                                                            |                                                                                                                                                                                                                                                         |
|                                                                                                                                                                                                                                                        | mysql> select * from tab_no_index where id = 2 for update;<br/>等待                                                                                                                                                                                       |

在如上表所示的例子中，看起来session_1只给一行加了排他锁，但session_2在请求其他行的排他锁时，却出现了锁等待！原因就是在没有索引的情况下，InnoDB只能使用表锁。当我们给其增加一个索引后，InnoDB就只锁定了符合条件的行，如下表所示。

创建tab_with_index表，id字段有普通索引：

```sql
create table tab_with_index(id int,name varchar(10)) engine=innodb;  
## Query OK, 0 rows affected (0.15 sec)  
alter table tab_with_index add index id(id);  
## Query OK, 4 rows affected (0.24 sec)  
## Records: 4  Duplicates: 0  Warnings: 0  
```

 InnoDB存储引擎的表在使用索引时使用行锁例子

| session_1                                                                                                                                                                                                                                                 | session_2                                                                                                                                                                                                                                                 |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)<br/>mysql> select * from tab_with_index where id = 1 ;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br/>1 row in set (0.00 sec) | mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)<br/>mysql> select * from tab_with_index where id = 2 ;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 2  \| 2  \|<br/>+------+------+<br/>1 row in set (0.00 sec) |
| mysql> select * from tab_with_index where id = 1 for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br/>1 row in set (0.00 sec)                                                             |                                                                                                                                                                                                                                                           |
|                                                                                                                                                                                                                                                           | mysql> select * from tab_with_index where id = 2 for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 2  \| 2  \|<br/>+------+------+<br/>1 row in set (0.00 sec)                                                             |

2. 由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是如果是使用相同的索引键，是会出现锁冲突的。应用设计的时候要注意这一点。

在如下表所示的例子中，表tab_with_index的id字段有索引，name字段没有索引：

```sql
alter table tab_with_index drop index name;  
## Query OK, 4 rows affected (0.22 sec)  
## Records: 4  Duplicates: 0  Warnings: 0  
insert into tab_with_index  values(1,'4');  
## Query OK, 1 row affected (0.00 sec)  
select * from tab_with_index where id = 1;  
+------+------+  
| id   | name |  
+------+------+  
| 1    | 1    |  
| 1    | 4    |  
+------+------+  
2 rows in set (0.00 sec)  
```

 InnoDB存储引擎使用相同索引键的阻塞例子

| session_1                                                                                                                                                                                                    | session_2                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)                                                                                                                                            | mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)                                                                           |
| mysql> select * from tab_with_index where id = 1 and name = '1' for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>+------+------+<br/>1 row in set (0.00 sec) |                                                                                                                                             |
|                                                                                                                                                                                                              | 虽然session_2访问的是和session_1不同的记录，但是因为使用了相同的索引,所以需要等待锁：<br/>mysql> select * from tab_with_index where id = 1 and name = '4' for update;<br/>等待 |

3. 当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。

在如下表所示的例子中，表tab_with_index的id字段有主键索引，name字段有普通索引：

```sql
alter table tab_with_index add index name(name);  
## Query OK, 5 rows affected (0.23 sec)  
## Records: 5  Duplicates: 0  Warnings: 0  
```

 InnoDB存储引擎的表使用不同索引的阻塞例子

| session_1                                                                                                                                                                                                         | session_2                                                                                                                                                                                                                                       |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)                                                                                                                                                 | mysql> set autocommit=0;<br/>Query OK, 0 rows affected (0.00 sec)                                                                                                                                                                               |
| mysql> select * from tab_with_index where id = 1 for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 1  \| 1  \|<br/>\| 1  \| 4  \|<br/>+------+------+<br/>2 rows in set (0.00 sec) |                                                                                                                                                                                                                                                 |
|                                                                                                                                                                                                                   | Session_2使用name的索引访问记录，因为记录没有被索引，所以可以获得锁：<br/>mysql> select * from tab_with_index where name = '2' for update;<br/>+------+------+<br/>\| id  \| name \|<br/>+------+------+<br/>\| 2  \| 2  \|<br/>+------+------+<br/>1 row in set (0.00 sec) |
|                                                                                                                                                                                                                   | 由于访问的记录已经被session_1锁定，所以等待获得锁。<br/>mysql> select * from tab_with_index where name = '4' for update;                                                                                                                                             |

4. 即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，**如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁**。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。

在下面的例子中，检索值的数据类型与索引字段不同，虽然MySQL能够进行数据类型转换，但却不会使用索引，从而导致InnoDB使用表锁。通过用explain检查两条SQL的执行计划，我们可以清楚地看到了这一点。

例子中tab_with_index表的name字段有索引，但是name字段是varchar类型的，如果where条件中不是和varchar类型进行比较，则会对name进行类型转换，而执行的全表扫描。

```sql
alter table tab_no_index add index name(name);  
## Query OK, 4 rows affected (8.06 sec)  
## Records: 4  Duplicates: 0  Warnings: 0  
explain select * from tab_with_index where name = 1 \G  
*************************** 1. row ***************************  
           id: 1  
  select_type: SIMPLE  
        table: tab_with_index  
         type: ALL  
possible_keys: name  
          key: NULL  
      key_len: NULL  
          ref: NULL  
         rows: 4  
        Extra: Using where  
1 row in set (0.00 sec)  
mysql> explain select * from tab_with_index where name = '1' \G  
*************************** 1. row ***************************  
           id: 1  
  select_type: SIMPLE  
        table: tab_with_index  
         type: ref  
possible_keys: name  
          key: name  
      key_len: 23  
          ref: const  
         rows: 1  
        Extra: Using where  
1 row in set (0.00 sec)  
```

#### 4. 间隙锁(Gap Lock)与Next -Key锁

**间隙锁**基于`非唯一索引`，它`锁定一段范围内的索引记录`。**间隙锁**基于下面将会提到的`Next-Key Locking` 算法，请务必牢记：**使用间隙锁锁住的是一个区间，而不仅仅是这个区间中的每一条数据**。

```sql
SELECT * FROM table WHERE id BETWEN 1 AND 10 FOR UPDATE;
```

即所有在`（1，10）`区间内的记录行都会被锁住，所有id 为 2、3、4、5、6、7、8、9 的数据行的插入会被阻塞，但是 1 和 10 两条记录行并不会被锁住。
除了手动加锁外，在执行完某些 SQL 后，InnoDB 也会自动加**间隙锁**，这个我们在下面会提到。

**临键锁（Next-Key Locks）**
Next-Key 可以理解为一种特殊的**间隙锁**，也可以理解为一种特殊的**算法**。通过**临建锁**可以解决`幻读`的问题。 每个数据行上的`非唯一索引列`上都会存在一把**临键锁**，当某个事务持有该数据行的**临键锁**时，会锁住一段**左开右闭区间**的数据。需要强调的一点是，`InnoDB` 中`行级锁`是基于索引实现的，**临键锁**只与`非唯一索引列`有关，在`唯一索引列`（包括`主键列`）上不存在**临键锁**。

假设有如下表：
**MySql**，**InnoDB**，**Repeatable-Read**：table(id PK, age KEY, name)

| id  | age | name  |
| --- | --- | ----- |
| 1   | 10  | Lee   |
| 3   | 24  | sower |
| 5   | 32  | cp    |
| 7   | 45  | fansw |

该表中 `age` 列潜在的`临键锁`有：
(-∞, 10],
(10, 24],
(24, 32],
(32, 45],
(45, +∞],
在`事务 A` 中执行如下命令：

```sql
-- 根据非唯一索引列 UPDATE 某条记录 
UPDATE table SET name = Vladimir WHERE age = 24; 
-- 或根据非唯一索引列 锁住某条记录 
SELECT * FROM table WHERE age = 24 FOR UPDATE; 
```

不管执行了上述 SQL 中的哪一句，之后如果在`事务 B` 中执行以下命令，则该命令会被阻塞：

```sql
INSERT INTO table VALUES(100, 26, 'Ezreal'); 
```

很明显，`事务 A` 在对 `age` 为 24 的列进行 UPDATE 操作的同时，也获取了 `(24, 32]` 这个区间内的临键锁。
不仅如此，在执行以下 SQL 时，也会陷入阻塞等待：

```sql
INSERT INTO table VALUES(100, 30, 'Ezreal'); 
```

那最终我们就可以得知，在根据`非唯一索引` 对记录行进行 `UPDATE \ FOR UPDATE \ LOCK IN SHARE MODE` 操作时，InnoDB 会获取该记录行的 `临键锁` ，并同时获取该记录行下一个区间的`间隙锁`。
即`事务 A`在执行了上述的 SQL 后，最终被锁住的记录区间为 `(10, 32)`。
**总结:**

1. **InnoDB** 中的`行锁`的实现依赖于`索引`，一旦某个加锁操作没有使用到索引，那么该锁就会退化为`表锁`。
2. **记录锁**存在于包括`主键索引`在内的`唯一索引`中，锁定单条索引记录。
3. **间隙锁**存在于`非唯一索引`中，锁定`开区间`范围内的一段间隔，它是基于**临键锁**实现的。
4. **临键锁**存在于`非唯一索引`中，该类型的每条记录的索引上都存在这种锁，它是一种特殊的**间隙锁**，锁定一段`左开右闭`的索引区间。

------

另外一个网站的举例：

举例来说，假如emp表中只有101条记录，其empid的值分别是 1,2,...,100,101，下面的SQL：

```sql
Select * from  emp where empid > 100 for update;
```

是一个范围条件的检索，InnoDB不仅会对符合条件的empid值为101的记录加锁，也会对empid大于101（这些记录并不存在）的“间隙”加锁。

InnoDB使用间隙锁的目的，一方面是为了防止幻读，以满足相关隔离级别的要求，对于上面的例子，要是不使用间隙锁，如果其它事务插入了empid大于100的任何记录，那么本事务如果再次执行上述语句，就会发生幻读；另外一方面，是为了满足其恢复和复制的需要。

很显然，在使用范围条件检索并锁定记录时，InnoDB这种加锁机制会阻塞符合条件范围内键值的并发插入，这往往会造成严重的锁等待。因此，在实际应用开发中，尤其是并发插入比较多的应用，我们要尽量优化业务逻辑，尽量使用相等条件来访问更新数据，避免使用范围条件。

还要特别说明的是，InnoDB除了通过范围条件加锁时使用间隙锁外，如果使用相等条件请求给一个不存在的记录加锁，InnoDB也会使用间隙锁！

在如下表所示的例子中，假如emp表中只有101条记录，其empid的值分别是1,2,......,100,101。

​                                  *InnoDB存储引擎的间隙锁阻塞例子*

| session_1                                                                                                                                                                                                                                                    | session_2                                                                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| mysql> select @@tx_isolation;<br/>+-----------------+<br/>\| @@tx_isolation \|<br/>+-----------------+<br/>\| REPEATABLE-READ \|<br/>+-----------------+<br/>1 row in set (0.00 sec)<br/>mysql> set autocommit = 0;<br/>Query OK, 0 rows affected (0.00 sec) | mysql> select @@tx_isolation;<br/>+-----------------+<br/>\| @@tx_isolation \|<br/>+-----------------+<br/>\| REPEATABLE-READ \|<br/>+-----------------+<br/>1 row in set (0.00 sec)<br/>mysql> set autocommit = 0;<br/>Query OK, 0 rows affected (0.00 sec) |
| 当前session对不存在的记录加for update的锁：<br/>mysql> select * from emp where empid = 102 for update;<br/>Empty set (0.00 sec)                                                                                                                                           |                                                                                                                                                                                                                                                              |
|                                                                                                                                                                                                                                                              | 这时，如果其他session插入empid为102的记录（注意：这条记录并不存在），也会出现锁等待：<br/>mysql>insert into emp(empid,...) values(102,...);<br/>阻塞等待                                                                                                                                            |
| Session_1 执行rollback：mysql> rollback;<br/>Query OK, 0 rows affected (13.04 sec)                                                                                                                                                                              |                                                                                                                                                                                                                                                              |
|                                                                                                                                                                                                                                                              | 由于其他session_1回退后释放了Next-Key锁，当前session可以获得锁并成功插入记录：<br/>mysql>insert into emp(empid,...) values(102,...);<br/>Query OK, 1 row affected (13.35 sec)                                                                                                           |

#### 5.插入意向锁

> 前言：
> 
> 在讲解之前，先来思考一个问题——假设有用户表结构如下：
> **MySql**，**InnoDB**，**Repeatable-Read**：users(id PK, name, age KEY)

| id  | name | age |
| --- | ---- | --- |
| 1   | Mike | 10  |
| 2   | Jone | 20  |
| 3   | Tony | 30  |

首先事务A插入了一行数据，并且没有Commit

```sql
INSERT INTO users SELECT 4, 'Bill', 15;
```

随后事务B试图插入一行数据

```sql
INSERT INTO users SELECT 5, 'Louis', 16;
```

请问：

1. 使用了什么锁？
2. 事务B是否会被事务A阻塞？

插入意向锁（Insert Intention Locks）

`插入意向锁`是在插入一条记录行前，由 **INSERT** 操作产生的一种`间隙锁`。该锁用以表示插入**意向**，当多个事务在**同一区间**（gap）插入**位置不同**的多条数据时，事务之间**不需要互相等待**。假设存在两条值分别为 4 和 7 的记录，两个不同的事务分别试图插入值为 5 和 6 的两条记录，每个事务在获取插入行上独占的（排他）锁前，都会获取（4，7）之间的`间隙锁`，但是因为数据行之间并不冲突，所以两个事务之间并**不会产生冲突**（阻塞等待）。

总结来说，`插入意向锁`的特性可以分成两部分：

1. `插入意向锁`是一种特殊的`间隙锁` —— `间隙锁`可以锁定**开区间**内的部分记录。
2. `插入意向锁`之间互不排斥，所以即使多个事务在同一区间插入多条记录，只要记录本身（`主键`、`唯一索引`）不冲突，那么事务之间就不会出现**冲突等待**。

需要强调的是，虽然`插入意向锁`中含有`意向锁`三个字，但是它并不属于`意向锁`而属于`间隙锁`，因为`意向锁`是**表锁**而`插入意向锁`是**行锁**。
现在我们可以回答开头的问题了：

1. 使用`插入意向锁`与`记录锁`。
2. `事务 A` 不会阻塞`事务 B`。

为什么不用间隙锁
如果只是使用普通的`间隙锁`会怎么样呢？还是使用我们文章开头的数据表为例：

首先`事务 A` 插入了一行数据，并且没有 `commit`：

```sql
INSERT INTO users SELECT 4, 'Bill', 15; 
```

此时 `users` 表中存在**三把锁**：

1. id 为 4 的记录行的`记录锁`。
2. age 区间在（10，15）的`间隙锁`。
3. age 区间在（15，20）的`间隙锁`。

最终，`事务 A` 插入了该行数据，并锁住了（10，20）这个区间。
随后`事务 B` 试图插入一行数据：

```sql
INSERT INTO users SELECT 5, 'Louis', 16; 
```

因为 16 位于（15，20）区间内，而该区间内又存在一把`间隙锁`，所以`事务 B` 别说想申请自己的`间隙锁`了，它甚至不能获取该行的`记录锁`，自然只能乖乖的等待 `事务 A` 结束，才能执行插入操作。
很明显，这样做事务之间将会频发陷入**阻塞等待**，**插入的并发性**非常之差。这时如果我们再去回想我们刚刚讲过的`插入意向锁`，就不难发现它是如何优雅的解决了**并发插入**的问题。

**总结：**

1. **MySql InnoDB** 在 `Repeatable-Read` 的事务隔离级别下，使用`插入意向锁`来控制和解决并发插入。
2. `插入意向锁`是一种特殊的`间隙锁`。
3. `插入意向锁`在锁定区间相同但记录行本身不冲突的情况下互不排斥。