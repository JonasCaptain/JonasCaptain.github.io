---
layout: "@layouts/ArticleLayout.astro"
title: MySQL存储引擎
description: 主要摘抄至出版书籍
date: 2023-02-20 10:00:00
tags:
  - MySQL
  - StorageEngine
---

## 存储引擎

### 1.MySQL常见的三种存储引擎(InnoDB、MyISAM，MEMORY)的区别？

1. InnoDB支持事务，MyISAM不支持。事务是一种高级的处理方式，如在一些列增删改中只要那个出错还可以回滚还原，而MyISAM就不行。
2. MyISAM适合查询以及插入为主的应用。
3. InnoDB适合频繁修改以及涉及到安全性较高的应用。
4. InnoDB支持外键，MyISAM不支持。
5. 从MySQL5.5之后，InnoDB是默认引擎。
6. InnoDB不支持FullText类型的索引。
7. InnoDB中不保存表的行数，如SELECT COUNT() from table时，InnoDB需要扫描一遍整个表来计算有多少行，但是MyISAM只要简单的读出保存好的行数即可。注意的是，当count()语句包含where条件时，MyISAM也需要扫描整个表。
8. 对于自增长的字段，InnoDB中必须包含只有该字段的索引，但是在MyISAM表中，可以和其他字段一起建立联合索引。
9. DELETE FROM table时，InnoDB不会重新建立表，而是一行一行的删除，效率非常慢，MyISAM则会重新建表。
10. InnoDB支持行锁(某些情况还是锁整表，如update table set a = 1 where user like '%lee%')。

### 2.MySQL存储引擎中的MyISAM与InnoDB如何选择？

MySQL实际上有很多引擎，每个引擎都有各自的优缺点，可以择优选择使用：MyISAM，InnoDB，MERGE、MEMORY（HEAP）、BDB（BerkeleyDB）、EXAMPLE、FEDERATED、ARCHIVE、CSV、BLACKHOLE。

虽然引擎种类很多，但常用的就是InnoDB和MyISAM

1. InnoDB会支持一些数据库高级功能，如事务功能和行级锁，MyISAM不支持。
2. MyISAM的性能更优，占用的存储空间少，所以选择何种引擎，视具体情况而定。

MEMORY是MySQL汇总一类特殊的存储引擎，它使用存储在内存中的内容来创建表，而且数据全部放在内存中。每个基于MEMORY存储引擎的表，实际对应一个磁盘文件。改文件的文件名与表名相同，类型为frm类型。该文件中只存储表的结构。而其数据文件，都是存储在内存中，这样有利于数据的快速处理，提高整个存储引擎的表的使用。如果不需要了，可以释放内存，甚至删除不需要的表。

### 3.MySQL的MyISAM与InnoDB在事务、锁级别各自的使用场景？

事务处理方面：

* MyISAM强调的是性能，每次查询具有原子性，其执行效率比InnoDB快，但不支持事务
* InnoDB提供事务支持，外键等高级数据库功能。具有事务提交(commit)、事务回滚(rollback)和崩溃修复能力(crash recovery capabilites)的事务安全型表

锁级别：

* MyISAM只支持表级锁，用户在操作MyISAM表时，SELECT、UPDATE、DELETE、INSERT语句都会给表自动加锁，如果加锁以后的表满足INSERT并发的情况下，可以在表的尾部插入新的数据。
* InnoDB支持事务后行级锁，是InnoDB最大的特色。行锁大幅度提高了多用户并发操作的性能。但是InnoDB的行数，只是在Where的主键是有效的前提下，非主键的Where都会锁全表的。
