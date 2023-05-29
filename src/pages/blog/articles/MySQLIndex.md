---
layout: "@layouts/ArticleLayout.astro"
title: MySQL索引
description: 主要摘抄至出版书籍
date: 2023-02-20 10:00:00
tags:
  - MySQL
  - Index
  - 索引
---

## 索引

### 简介

索引有两种，一种是BTree索引，一种是Hash索引

### 分类

1. 普通索引：不添加任何限制条件
2. 唯一性索引：使用Unique参数，确保该字段中所有的value具有唯一性
3. 全文索引：使用FullText参数，只能创建在Char、VarChar、Text类型的字段上，只有MyISAM存储引擎支持全文索引
4. 单列索引：在一个字段上建立的普通索引，唯一性索引或全文索引
5. 多列索引：在多个字段上建立的普通索引，唯一性索引或全文索引
6. 空间索引：使用Spatial参数，只有MyISAM存储引擎支持空间索引，必须建立在空间数据类型上，且必须非空

#### 索引的设计原则

1. 选择唯一性索引

2. 为经常需要排序、分组和联合操作的字段建立索引</br>
   `如：order by、group by、distinct、union等操作的字段，特别是排序`

3. 为常作为查询条件的字段建立索引

4. 限制索引的数目（避免过多的浪费空间）

5. 尽量使用数据量少的索引

6. 尽量使用前缀来索引</br>
   `如：索引Text类型字段的前N个字符`

7. 删除不再使用或者很少使用的索引

#### 创建索引

三种方式：

* 创建表时创建索引(create table xxx 附带)
* 已经存在的表上创建索引(create index)
* 使用alter table语句来创建索引(alter table add index)

#### 创建表时创建索引

```sql
## 普通索引
create table index1(id int,
                    name varchar(20),
                       sex boolean,
                       index(id));
## 唯一性索引
create table index2(id int unique,
                       name varchar(20));

create table index2(id int,
                       name varchar(20),
                    unique index index2_id(id ASC));

## 全文索引
create table index3(id int,
                   info varchar(20),
                   fulltext index index3_info(info))
                   engine=MyISAM;
## 创建单列索引
create table index4(id int,
                       subject varchar(30),
                       index index4_st(subject(10)));
                       ## 只索引subject前10个字符
## 创建多列索引
create table index5(id int,
                       name varchar(20),
                       sex char(4),
                       index index5_ns(name, sex));
explain select * from index5 where name = '123'\G;
explain select * from index5 where name = '123' and sex = 'N'\G;

## 创建空间索引
create table index6(id int,
                       space geometry not null,
                   spatial index index6_sp(space))
                   engine=MyISAM;
```

#### 在已经存在的表上创建索引

```sql
CREATE [UNIQUE|FULLTEXT|SPATIAL] INDEX 索引名 ON 表名(属性名[(长度)] [ASC|DESC]);

## 创建普通索引
create index index7_id on index7(id);
## 创建唯一性索引
create unique index8_id on index8(course_id);
## 创建全文索引
create fulltext index index9_info on index9(info);
## 创建单列索引
create index index10_addr on index10(address(4));
## 创建多列索引
create index index11_na on index11(name, address);
## 创建空间索引
create spatial index12_line on index12(line);
```

#### 用alter table语句创建索引

```sql
ALTER TABLE 表名 ADD [UNIQUE|FULLTEXT|SPATIAL] INDEX 索引名 (属性名[(长度)] [ASC|DESC]);

## 创建普通索引
alter table index13 add index index13_name(name(20));
## 创建唯一性索引
alter table index14 add unique index index14_id(course_id);
## 创建全文索引
alter table index15 add fulltext index index15_info(info);
## 创建单列索引
alter table index16 add index index16_addr(address(4));
## 创建多列索引
alter table index17 add index index17_na(name, address);
## 创建空间索引
alter table index18 add spatial index index18_line(line);
```

#### 删除索引

```sql
drop index 索引名 on 表名;
drop index id on index1;
```

### B+Tree索引和Hash索引的区别

#### 特点

* Hash索引结构的特殊性，其检索效率非常高，索引的检索可以一次定位
* B+Tree索引需要从根节点到枝节点，最后才能访问到叶节点这样多次的**IO**访问

#### 为什么不都用Hash索引而使用B+Tree索引？

1. Hash索引仅仅能满足“=”、“IN”和“”查询，不能使用范围查询，因为经过相应的Hash算法处理之后的Hash值的大小关系，并不能保证和Hash运算前完全一样
2. Hash索引无法被用来避免数据的排序操作，因为Hash值的大小关系并不一定和Hash运算前的键值完全一样
3. Hash索引不能利用部分索引键查询，对于组合索引，Hash索引在计算Hash值的时候，是组合索引键合并后再一起计算Hash值，而不是单独的计算Hash值，所以通过组合索引的前面一个或几个索引键进行查询的时候，Hash索引也无法被利用
4. Hash索引在任何时候都不能避免表扫描，由于不同索引键存在相同Hash值，所以即使满足某个Hash键值数据的记录条数，也无法从Hash索引中直接完成查询，还是要回表查询数据
5. Hash索引遇到大量Hash值相等的情况后性能并不一定就会比B+Tree索引高

#### 补充

1. MySQL中，只有HEAP/MEMORY引擎才显式支持Hash索引
2. 常用的**InnoDB**引擎中默认使用的是B+Tree索引，它会实时监控表上索引的使用情况，如果认为建立Hash索引可以提高查询效率，则自动在内存中的“自适应Hash索引缓冲区”建立Hash索引(在InnoDB中默认开启自适应Hash索引)，通过观察搜索模式，MySQL会利用index key的前缀建立Hash索引，入股过一个表几乎大部分都在缓冲池中，那么建立一个Hash索引能够加快等值查询
3. 如果是等值查询，那么Hash索引明显有绝对优势，因为只需要经过一次算法即可找到相应的键值；当然了，这个前提是，键值都是唯一的。如果键值不是唯一的，就需要先找到该键所在的位置，然后再根据链表往后扫描，直到找到相应的数据
4. 如果是范围查询检索，这时候Hash索引就毫无用武之地了，因为原先是有序的键值，经过Hash算法后，有可能变成不连续的了，就没办法再利用索引完成范围查询检索；同理Hash索引没办法利用索引完成排序，以及 like 'xx%'这样的部分模糊查询(这种部分模糊查询，本质上也是范围查询)
5. Hash索引不支持多列联合索引的最左匹配规则
6. B+Tree索引的关键字检索效率比较平均，不像BTree那样波动幅度大，在有大量重复键值情况下，Hash索引的效率也是极低的，因为存在所谓的Hash碰撞问题
7. 在大多数场景下，都会有范围查询、排序、分组等查询特征，用B+Tree索引就可以了

#### BTree与B+Tree的区别

* BTree，每个节点存储Key和Data，所有节点组成这颗树，并且叶子节点指针为null，叶子节点不包含任何关键字信息
* B+Tree，所有的叶子节点中包含了全部关键字的信息，及指向含有这些关键字记录的指针，且叶子节点本身依关键字的大小自小而大的顺序链接，所有的非终端节点可以看成是索引部分，节点中仅含有其子树根节点中最（或最小）关键字。（而BTree的非终节点也包含需要查找的有效信息）

#### 为什么说B+Tree比BTree更适合实际应用中操作系统的文件索引和数据库索引？

* B+的磁盘读写代价更低
  
  B+的内部节点并没有指向关键字具体信息的指针。因此其内部节点相对BTree更小。如果把所有同一内部节点的关键字存放在同一块盘中，那么盘所能容纳的关键字数量也越多。一次性读入内存中的需要查找的关键字也就越多。相对来说IO读写次数也就降低了。

* B+Tree的查询效率更加稳定
  
  由于非终节点并不是最终指向文件内容的节点，而只是叶子节点中的关键字的索引。所以任何关键字的查找必须走一条从根节点到叶子节点的路。所有关键字查询的路径长度相同，导致每一个数据的查询效率相当。

### 聚簇索引和非聚簇索引的区别

1. 聚簇索引(clustered index):
   
   聚簇索引表记录的排列顺序和索引的排列顺序一致，所以查询效率快，只要找到第一个索引值记录，其余就连续性的记录在物理也一样连续存放。聚簇索引的对应的缺点就是修改慢，因为为了保证表中记录的物理和索引顺序一致，在记录插入的时候，会对数据页重新排序。
   
   聚簇索引类似于新华字典总用拼音去查找汉字，拼音检索表的顺序都是按照a~z排列的，就想相同的逻辑顺序与物理顺序一样，当你需要查找a,ai两个读音的字，或是想一次寻找多个sha的同音字时，也许向后翻几页，或紧接着下一行就得到结果了。

2. 非聚簇索引(nonclustered index):
   
   非聚簇索引指定了表中记录的逻辑顺序，但是记录的物理和索引不一定一致，两种索引都采用B+Tree结构，非聚簇索引的叶子层并不和实际数据页相重叠，而采用叶子层包含一个指向表中的记录在数据页中的指针方式。非聚簇索引层次多，不会造成数据重排。
   
   非聚簇索引类似在新华字典上通过偏旁部首来查询汉字，检索表也是按照横、竖、撇来排列的，但是由于正文中是a~z的拼音顺序，所以就类似于逻辑地址与物理地址的不对应。同时适用的情况就在于分组，大数目的不同值，频繁更新的列中，这些情况即不适合聚簇索引。

**根本区别**：

聚簇索引和非聚簇索引的根本区别是表记录的排列顺序和与索引的排列顺序是否一致。
