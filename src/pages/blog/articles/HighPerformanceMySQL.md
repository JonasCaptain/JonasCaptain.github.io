---
layout: "@layouts/ArticleLayout.astro"
title: 高性能Mysql
description: 主要摘抄至出版书籍
date: 20 Feb 2023
tags:
  - MySQL
  - Performance
  - 性能
---


## 优化特定类型的查询

### 1.优化count()查询

作用：

1. 统计某个列值的数量。要求列值是非空的（不统计NULL值）。如果在count()的括号中指定了列或列的表达式，则统计的就是这个表达式有值的结果数。
2. 统计行数。当MySQL确认括号内的表达式值不可能为空时，实际上就是在统计行数。最简单的就是在使用count(*)时，这种情况下通配符\*并不会像我们猜想的那样扩展成所有的列，实际上，它会忽略所有的列而直接统计所有的行数。

简单的优化：

```sql
## 如何快速查找到所有ID大于5的城市
select count(*) from world.City where id > 5;
## 这样做的结果是该查询需要扫描4097行数据，如果将条件反转一下，先查找ID小于5的城市数
## 然后用总城市数一减就能得到同样的结果，却可以将扫描的行数减少到5行以内
select (select count(*) from world.City) - count(*) from world.City where ID <=5;
## 这样做可以大大减少需要扫描的行数，是因为在查询优化阶段会将其中的子查询直接当做一个常数来处理。
```

有时候会遇到这样的问题，在同一个查询中统计同一个列中不同值的数量，以减少查询的语句量。假如：假设可能需要同归一个查询返回各种不同颜色的商品数量，此时不能使用or语句。

```sql
select count(color='blue' or color='red') from items;
## 上面这样做就无法区分不同颜色的商品数量，也不能在where条件中指定颜色
select count(*) from items where color='blue' and color='red';
## 因为颜色条件是互斥的

## 下面查询可以在一定程度上解决这个问题
select sum(if(color='blue', 1, 0)) as blue, sum(if(color='red', 1, 0)) as red from items;
select sum(color='blue') as blue, sum(color='red') as red from items;
## 或者
select count(color='blue' or null) as blue, count(color='red' or null) as red from items;
```

使用近似值：

有时候某些业务并不要求完全精确的count值，此时可以用近似值来替代。explain出来的优化器估算的行数就是一个不错的近似值，执行explain并不需要真正地去执行查询，所以成本很低。很多时候，精确计算的成本非常高，而计算近似值则非常简单。

曾今有一个客户希望统计他的网站的当前活跃用户数是多少，这个活跃用户数保存在缓存中，过期时间为30分钟，所以每个30分钟需要重新计算并放入缓存。因此这个活跃用户数本身就不是精确值，所以使用近似值替代是可以接受的。另外，如果要精确统计在线人数，通常where条件是会很复杂，一方面需要剔除当前非活跃用户，另一方面还要剔除系统中某些特定ID的“默认”用户，去掉这些约束条件对总数的影响很小，但却可能很好地提升该查询的性能。更进一步的优化则可以尝试删除distinct这样的约束来避免文件排序。这样重写过的查询要比原来的精确统计的查询快很多，而返回的结果则几乎相同。

更复杂的优化：

通常来说，count()都需要扫描大量的行才能获得精确地结果，因此很难优化。除了之前的方法，在MySQL层面还能做的就只有索引覆盖扫描了。如果这不够，就需要考虑修改应用架构，可以增加汇总表或者增加类似memcached这样的外部缓存系统。

### 2.优化关联查询

* 确保ON或者USING子句中的列上有索引。在创建索引的时候要考虑到关联的顺序。当表A和表B用列C关联时，如果优化器的关联顺序是B、A，那么就不需要在B表的对应列上建立索引。没有用到的索引只会带来额外的负担。一般来说，除非其他理由，否则只需要在关联顺序中的第二个表的相应列上创建索引。
* 确保任何的GROUP BY和ORDER BY中的表达式只涉及到一个表中的列，这样MySQL才有可能使用索引来优化这个过程。
* 当升级MySQL的时候需要注意：关联语法、运算符优先级等其他可能会发生变化的地方。因为以前是普通关联的地方可能会变成笛卡尔积，不同类型的关联可能会生成不同的结果等。

### 3.优化子查询

关于子查询优化我们给出的最重要的优化建议就是尽可能使用关联查询代替，至少当前的MySQL版本需要这样。

### 4.优化GROUP BY 和DISTINCT

在很多场景下，MySQL都使用同样的办法优化这两种查询，事实上，MySQL优化器会在内部处理的时候相互转换这两类查询。它们都可以使用索引来优化，这也是最有效的优化办法。

在MySQL中，当无法使用索引的时候，GROUP BY 使用两种策略来完成：使用临时表或者文件排序来做分组。对于任何查询语句，这两策略的性能都有可以提升的地方。可以通过使用提示SQL_BIG_RESULT 和 SQL_SMALL_RESULT 来让优化器按照你希望的方式运行。

如果需要对关联查询做分组(GROUP BY)，并且是按照查找表中的某个列进行分组，那么通常采用查找表的标识列分组的效率会比其他列更高。

```sql
## 例如下面的查询效率不会很好
select actor.first_name, actor.last_name, count(*)
from sakila.film_actor
    inner join sakila.actor using(actor_id)
group by acotr.first_name, actor.last_name;

## 假如按照下面的写法效率会更高
select actor.first_name, actor.last_name, count(*)
from sakila.film_actor
    inner join sakila.actor using(actor_id)
group by film_actor.actor_id;
```

这个查询利用了actor的id直接相关的特点，因此改写后的结果不受影响，但显然不是所哟的关联语句的分组查询都可以改写成在select中直接使用非分组列的形式的。甚至可能会在服务器上设置SQL_MODE来禁止这样的写法。如果是这样，也可以通过MIN()或者MAX()函数来绕过这种限制，但一定要清楚，select后面出现的非分组列一定是直接依赖分组列，并且在每个组内的值是唯一的，或者是业务上根本不在乎这个值具体是什么：

```sql
select min(actor.first_name), max(actor.last_name), ....;
```

**优化GROUP BY WITH ROLLUP**
分组查询的一个变种就是要求MySQL对返回的分组结果再做一次超级聚合。可以使用with rollup子句来实现这种逻辑，但可能会不够优化。可以通过explain来观察其执行计划，特别要注意分组是否是通过文件排序或者临时表实现的。然后再去掉with rollup子句看执行计划是否相同。

### 5.优化LIMIT分页

在偏移量非常大的时候，例如可能是 limit 10000,20这样的查询，这是MySQL需要查询10020条记录然后只返回最后的20条，前面的10000条记录都将被抛弃。如果所有的页面被访问的频率都相同，那么这样的查询平均需要访问半个表的数据。要优化这种查询，要么是在页面中限制分页的数量，要么是优化大偏移量的性能。

优化此类分页查询的一个最简单的办法就是尽可能的使用索引覆盖扫描，而不是查询所有的列。然后根据需要做一次关联操作再返回所需的列。对于偏移量很大的时候，这样做的效率会提升非常大。

```sql
select film_id, description, from sakila.film order by title limit 50, 5;
## 如果sakila.film表非常大，这个查询最好改写成下面的样子
select film.film_id, film.description
from sakila.film
    inner join(
        select film_id from sakila.film
        order by title limit 50, 5
    ) as lim using(film_id);
```

这里的“延迟关联”将大大提升查询效率，它让MySQL扫描尽可能少的页面，获取需要访问的记录后再根据关联列会原表查询需要的所有列。这个技术也可以用于优化关联查询中的limit子句。

有时候也可以将limit查询转换为已知位置的查询，让MySQL通过范围扫描获得到对应的结果。例如，如果在一个位置列上有索引，并且预先计算出了边界值，上面的查询就可以改写为：

```sql
select film_id, description from sakila.film
where position between 50 and 54 order by position;
```

对数据进行排名的问题也与此类似，但往往还会同时和GROUP BY混合使用。在这种情况下通常都需要预先计算并存储排名信息。

limit和offset问题，其实是offset的问题，它会导致MySQL扫描大量不需要的行然后抛弃掉。如果可以使用书签记录上次取数据的位置，那么下次就可以直接从该书签记录的位置开始扫描，这样就可以避免使用offset。

```sql
select * from sakila.rental order by rental_id desc limit 20;
## 假设上面的查询返回的是主键为16049到16030的记录，那么下一页查询就可以从16030这个点开始
select * from sakila.rental 
where rental_id < 16030 order by rental_id desc limit 20;
```

此类改写的好处是无论翻页到多么靠后，其性能都会很好。
