---
layout: "@layouts/ArticleLayout.astro"
title: MySQL查询分类
description: 主要摘抄至出版书籍
date: 2023-03-20T00:00:00Z
tags:
  - MySQL
  - Query
  - 查询分类
---

## 数据库查询分类

### 多表查询

* 内连接

> 用比较运算符根据每个表共有的列的值匹配两个表中的行(=或>,<)

```sql
select
        d.Good_ID ,
        d.Classify_ID,
        d.Good_Name
        from
        Commodity_list d
        inner join commodity_classification c
        on d.Classify_Description=c.Good_kinds_Name
```

得到的满足某一条件的是A，B内部的数据；正因为得到的是内部共有数据，所以连接方式称为内连接

![内连接](/mysql/inner_join.png)

* 外连接之左连接

```sql
select
        *
        from
        commodity_classification c
        left join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
```

![左连接](/mysql/left_join.png)

首先是left_join左表的数据全部罗列，然后有满足条件的右表数据都会全部罗列出来，不满足条件的右表记录以NULL值显示。

**左连接升级**：left join 或者 left outer join(等同于left join) + where B.column is null

```sql
select
        *
        from
        commodity_classification c
        left join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
        where d.Classify_Description is null
```

![左外连接](/mysql/left_join_exclusive_null.png)

* 外连接之右连接

```sql
select
        *
        from
        commodity_classification c
        right join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
```

![右连接](/mysql/right_join.png)

与左连接相反，首先是右表数据全部罗列，然后又满足条件的左表数据都会全部罗列出来，不满足条件的左表记录以NULL值显示。

**右连接升级**：right join 或者 right outer join(等同于right join) + where A.column is null

![右外连接](/mysql/right_join_exclusive_null.png)

* 外连接之全外连接

full join ，mysql不支持，但是可以用left join union right join代替

```sql
select
        *
        from
        commodity_classification c
        left join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
        union
select
        *
        from
        commodity_classification c
        right join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
```

这种场景下得到的是满足某一条件的公共记录，和独有记录，union是合并的意思，所以这种方式的查询结果就是A、B的全集

![全连接](/mysql/full_join.png)

**全外连接升级**：

```sql
select
        *
        from
        commodity_classification c
        left join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
        where d.Classify_Description is null
        union
select
        *
        from
        commodity_classification c
        right join commodity_list d
        on d.Classify_Description=c.Good_kinds_Name
         where c.Good_kinds_Name is null
```

这种场景下得到的是A、B中不满足某一条件记录之和

![全外连接](/mysql/full_join_exclusive_null.png)

* 交叉连接

交叉连接返回左表中的所有行，左表中的每一行与右表中的所有行组合。交叉连接也称作笛卡尔积。

```sql
## 第一种方式，不指定where子句，得到的结果就是两表相乘
select
        *
        from
        commodity_classification c
       cross join commodity_list d
## 指定where子句，会先生成笛卡尔积，再从里面挑选符合where条件的记录
select
        *
        from
        commodity_classification c
       cross join commodity_list d 
       where c.Good_kinds_Name=d.Classify_Description

## 第二种方式，不显式指定cross join，同样得到的也是笛卡尔积
select
        *
        from
        commodity_classification c,
        commodity_list d 
       where c.Good_kinds_Name=d.Classify_Description
```

* union与union all

```sql
/*
UNION 用于合并两个或多个 SELECT 语句的结果集，并消去表中任何重复行。
UNION 内部的 SELECT 语句必须拥有相同数量的列，列也必须拥有相似的数据类型。
同时，每条 SELECT 语句中的列的顺序必须相同.
*/
SELECT column_name FROM table1
UNION
SELECT column_name FROM table2
/*
默认地，UNION 操作符选取不同的值。如果允许重复的值，请使用 UNION ALL。
当 ALL 随 UNION 一起使用时（即 UNION ALL），不消除重复行
*/
SELECT column_name FROM table1
UNION ALL
SELECT column_name FROM table2
```
