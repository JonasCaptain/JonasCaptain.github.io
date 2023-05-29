---
layout: "@layouts/ArticleLayout.astro"
title: MySQL必知必会
description: 主要摘抄至出版书籍
date: 2023-02-20 00:00:00
tags:
  - 必知必会
  - MySQL
---

## 《MySQL必知必会》

#### 子查询(杂记)

`子查询总是从内向外处理，即先执行子查询，将得到的结果作为外部查询的筛选条件`

#### 联结(连接)

###### 内部联结

等值联结(queijoin)，基于两个表之间的相等测试，这种联结也称为内部联结。

```sql
## 常用写法
select vend_name, prod_name, prod_price from vendors, products where vendors.vend_id = products.vend_id order by vend_name, prod_name;

## 标准写法
select vend_name, prod_name, prod_price from vendors inner join products on vendors.vend_id = products.vend_id order by vend_name, prod_name;
```

`Mark: ANSI SQL规范首选Inner Join语法。此外，尽管使用where子句定义联结的确比较简单，但是使用明确的联结语法能够确保不会忘记联结条件，有时候这样做也能影响性能。`

###### 联结多个表

SQL对一条select语句中可以联结的表的数目没有限制。创建联结的基本规则也相同。

```sql
select prod_name, vend_name, prod_price, quantity from orderitems, products, vendors where products.vend_id = vendors.vend_id and orderitems.prod_id = products.prod_id and order_num = 20005;

+----------------+-------------+------------+----------+
| prod_name      | vend_name   | prod_price | quantity |
+----------------+-------------+------------+----------+
| .5 ton anvil   | Anvils R Us |       5.99 |       10 |
| 1 ton anvil    | Anvils R Us |       9.99 |        3 |
| TNT (5 sticks) | ACME        |      10.00 |        5 |
| Bird seed      | ACME        |      10.00 |        1 |
+----------------+-------------+------------+----------+
4 rows in set (0.00 sec)
```

`Mark：Mysql在运行时关联指定的每个表以处理联结。这种处理可能是非常耗费资源的，因此应该仔细，不要联结不必要的表。联结的表越多，性能下降越厉害。`

有时候子查询并不总是执行复杂select操作的最有效的方法

```sql
select cust_name, cust_contact 
from customers 
where cust_id in (
    select cust_id 
    from orders 
    where order_num in (
        select order_num 
        from orderitems 
        where prod_id = 'TNT2'));
## ===>
select cust_name, cust_contatct
from customers, orders, orderitems
where customers.cust_id = order.cust_id
    and orderitems.order_num = orders.order_num
    and prod_id = 'TNT2';
```

###### 自联结

```sql
select prod_id, prod_name
from products
where vend_id = (select vend_id
                from products
                where prod_id = 'DTNTR');
## ===>
select p1.prod_id, p1.prod_name
from products as p1, products as p2
where p1.vend_id = p2.vend_id
and p2.prod_id = 'DTNTR';
```

`Mark：自联结通常作为外部语句用来替代从相同表中检索数据时使用的子查询语句。虽然最终的结果是相同的，但有时候处理联结远比处理子查询快得多。应该试一下两种方法，以确定哪一种的性能更好。`

###### 自然联结

无论何时对表进行联结，应该至少有一个列出现在不止一个表中(被联结的列)。标注你的联结返回所有数据，甚至相同的列多次出现。自然联结排除多次出现，使每个列只返回一次。自然联结是这样一种联结，其中你只能选择那些唯一的列。这一般是通过对表使用通配符(select *)，对所有其他表的列使用明确的子集来完成的。

```sql
select c.*, o.order_num, o.order_date,
       oi.prod_id, oi.quantity, OI.item_price
from customer as c, order as o, orderitems as oi
where c.cust_id = o.cust_id
and oi.order_num = o.order_num
and prod_id = 'FB';
```

在这个例子中，通配符只对第一个表使用。所有其他列明确列出，所以没有重复的列被检索出来。

事实上，迄今为止我们建立的每个内部联结都是自然联结，很可能我们永远都不会用到不是自然联结的内部联结。

###### 外部联结

许多联结将一个表中的行与另一个表中行相关联。但有时候会需要包含没有关联行的那些行，这种类型的联结称为外部联结。

```sql
## 内部联结，仅检索出匹配条件的记录
select customers.cust_id, orders.order_num
from customers inner join orders
on customers.cust_id = orders.cust_id;
+---------+-----------+
| cust_id | order_num |
+---------+-----------+
|   10001 |     20005 |
|   10001 |     20009 |
|   10003 |     20006 |
|   10004 |     20007 |
|   10005 |     20008 |
+---------+-----------+
5 rows in set (0.00 sec)

## 外部联结，可以检索出未匹配上的记录
select customers.cust_id, orders.order_num
from customers left outer join orders
on customers.cust_id = orders.cust_id;
+---------+-----------+
| cust_id | order_num |
+---------+-----------+
|   10001 |     20005 |
|   10001 |     20009 |
|   10002 |      NULL |
|   10003 |     20006 |
|   10004 |     20007 |
|   10005 |     20008 |
+---------+-----------+
6 rows in set (0.00 sec)
```

在使用外部联结**outer join**语法时，必须使用**right**或**left**关键字指定包括其所有行的表(right指出的是outer join 右边的表，而left指出的是outer join左边的表)

#### 组合查询

多数SQL查询都只包含从一个或多个表中返回数据的单条select语句。Mysql允许执行多个查询(多条select语句)，并将结果作为单个查询结果集返回。这些组合查询通常称为并(Union)或复合查询(compound query)。

有两种情况，其中需要使用组合查询：

* 在单个查询中从不同的表返回类似结构的数据；
* 对单个表执行多个查询，按单个查询返回数据。

举例：假如需要价格小于等于5的所有物品的一个列表，而且还想包括供应商1001和1002生产的所有物品(不考虑价格)。

```sql
## 单条SQL
select vend_id, prod_id, prod_price from products where prod_price <= 5;
+---------+---------+------------+
| vend_id | prod_id | prod_price |
+---------+---------+------------+
|    1003 | FC      |       2.50 |
|    1002 | FU1     |       3.42 |
|    1003 | SLING   |       4.49 |
|    1003 | TNT1    |       2.50 |
+---------+---------+------------+
4 rows in set (0.00 sec)
select vend_id, prod_id, prod_price from products where vend_id in (1001, 1002);
+---------+---------+------------+
| vend_id | prod_id | prod_price |
+---------+---------+------------+
|    1001 | ANV01   |       5.99 |
|    1001 | ANV02   |       9.99 |
|    1001 | ANV03   |      14.99 |
|    1002 | FU1     |       3.42 |
|    1002 | OL1     |       8.99 |
+---------+---------+------------+
5 rows in set (0.00 sec)

## 组合查询
select vend_id, prod_id, prod_price from products where prod_price <= 5
union
select vend_id, prod_id, prod_price from products where vend_id in (1001, 1002);
+---------+---------+------------+
| vend_id | prod_id | prod_price |
+---------+---------+------------+
|    1003 | FC      |       2.50 |
|    1002 | FU1     |       3.42 |
|    1003 | SLING   |       4.49 |
|    1003 | TNT1    |       2.50 |
|    1001 | ANV01   |       5.99 |
|    1001 | ANV02   |       9.99 |
|    1001 | ANV03   |      14.99 |
|    1002 | OL1     |       8.99 |
+---------+---------+------------+
8 rows in set (0.00 sec)

## 作为参考，列出使用多条where子句而不是使用union的相同查询
select vend_id, prod_id, prod_price 
from products
where prod_price < =5
or vend_id in (1001, 1002);
```

此例中，使用union可能比使用where子句更为复杂，但对于更复杂的过滤条件，或者从多个表(而不是单个表)中检索数据的情形，使用union可能会使处理更简单。

###### union规则

* union必须由两条或两条以上的select语句组成，语句之间用关键字union分隔
* union中的每个查询必须包含相同的列、表达式或聚集函数
* 列数据类型必须兼容：类型不必完全相同，但必须是DBMS可以隐含地转换的类型

从前面的例子中，可以看出来，第一条SQL查询结果是4条记录，第二条SQL查询结果是5条记录，而使用了union关键字之后，返回了8条记录而不是9条。union从查询结果集中自动去重了，这是union的默认行为，但是如果想要返回所有匹配的行，可使用union all而不是union

```sql
select vend_id, prod_id, prod_price from products where prod_price <= 5
union all
select vend_id, prod_id, prod_price from products where vend_id in (1001, 1002);
```

`Mark：union几乎总是完成与多个where条件相同的工作，但是union all完成了where完成不了的工作，如果确实需要需要每个条件的匹配行全部出现(包含重复)，则必须使用union all。`

#### 全文本检索(全文索引)

> 想使用全文本检索功能，表所使用的存储引擎必须是MyISAM

使用全文本索引的语法要求：

```sql
Syntax:
MATCH (col1,col2,...) AGAINST (expr [search_modifier])

select note_text from productnotes where Match(note_text) Against('rabbit');
+----------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                            |
+----------------------------------------------------------------------------------------------------------------------+
| Customer complaint: rabbit has been able to detect trap, food apparently less effective now.                         |
| Quantity varies, sold by the sack load.
All guaranteed to be bright and orange, and suitable for use as rabbit bait. |
+----------------------------------------------------------------------------------------------------------------------+
```

使用Match()和Against()执行全文本搜索，其中Match()指定被搜索的列，Against()指定要使用的搜索表达式。传递给Match()的值必须与Fulltext()定义中的相同，如果指定多个列，则必须列出它们(而且次序正确)。

上面的例子也可以用Like语句改写

```sql
select note_text
from productnotes
where note_text like '%rabbit%';
+----------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                            |
+----------------------------------------------------------------------------------------------------------------------+
| Quantity varies, sold by the sack load.
All guaranteed to be bright and orange, and suitable for use as rabbit bait. |
| Customer complaint: rabbit has been able to detect trap, food apparently less effective now.                         |
+----------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

改写后的like子句，同样可以检索出两行，但次序不同(虽然并不总是出现这种情况)。

上述两条select语句都不包含order by子句。后者以不特别有用的顺序返回数据。前者(使用全文本搜索)返回以文本匹配的良好程度排序的数据。两个行都包含此词rabbit，但包含词rabbit作为第3个词的行的等级比作为第20个词的行高。这很重要。全文本搜索的一个重要部分就是对结果排序。具有较高等级的行先返回(因为这些行很可能是你真正想要的行)。

```sql
select note_text,
       Match(note_text) Against('rabbit') as rank
from productnotes;

+-----------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| note_text                                                                                                                                                 | rank               |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| Customer complaint:
Sticks not individually wrapped, too easy to mistakenly detonate all at once.
Recommend individual wrapping.                          |                  0 |
| Can shipped full, refills not available.
Need to order new can if refill needed.                                                                          |                  0 |
| Safe is combination locked, combination not provided with safe.
This is rarely a problem as safes are typically blown up or dropped by customers.         |                  0 |
| Quantity varies, sold by the sack load.
All guaranteed to be bright and orange, and suitable for use as rabbit bait.                                      | 1.5905543565750122 |
| Included fuses are short and have been known to detonate too quickly for some customers.
Longer fuses are available (item FU1) and should be recommended. |                  0 |
| Matches not included, recommend purchase of matches or detonator (item DTNTR).                                                                            |                  0 |
| Please note that no returns will be accepted if safe opened using explosives.                                                                             |                  0 |
| Multiple customer returns, anvils failing to drop fast enough or falling backwards on purchaser. Recommend that customer considers using heavier anvils.  |                  0 |
| Item is extremely heavy. Designed for dropping, not recommended for use with slings, ropes, pulleys, or tightropes.                                       |                  0 |
| Customer complaint: rabbit has been able to detect trap, food apparently less effective now.                                                              | 1.6408053636550903 |
| Shipped unassembled, requires common tools (including oversized hammer).                                                                                  |                  0 |
| Customer complaint:
Circular hole in safe floor can apparently be easily cut with handsaw.                                                                |                  0 |
| Customer complaint:
Not heavy enough to generate flying stars around head of victim. If being purchased for dropping, recommend ANV02 or ANV03 instead.   |                  0 |
| Call from individual trapped in safe plummeting to the ground, suggests an escape hatch be added.
Comment forwarded to vendor.                            |                  0 |
+-----------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
14 rows in set (0.00 sec)
```

这里，在select而不是where子句中使用Match()和Against()。这使所有行都被返回(因为没有where子句)。Match()和Against()用来建立一个计算列(别名为rank)，此列包含全文本搜索计算出的等级值。等级由Mysql根据行中词的数目、唯一词的数目、整个索引中词的总数以及包含该词的行的数目计算出来。正如所见，不包含词rabbit的行等级为0。确实包含词rabbit的两个行每行都有一个等级值，文本中词靠前的行的等级值比词靠后的行的等级值高。

这个例子有助于说明全文本搜索如何排除行(排除那些等级为0的行)，如何排序结果(按等级以降序排序)。

`Mark：如果指定多个搜索项，则包含多数匹配词的那些行将具有比包含较少词(或仅有一个匹配)的那些行高的等级值`

通过对比执行计划，全文本搜索比Like的速度要快得多

```sql
mysql> explain select note_text from productnotes where note_text like '%rabb
it%'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: productnotes
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 14
     filtered: 11.11
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

mysql> explain select note_text from productnotes where Match(note_text) Against('rabbit')\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: productnotes
   partitions: NULL
         type: fulltext
possible_keys: note_text
          key: note_text
      key_len: 0
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

mysql>
```

全文本搜索仅检索了一行，就找到了需要的行，使用Like则是全表扫描，当然这没有可比性，一个是通过where顾虑，一个是走索引，但是这种涉及到文本检索的情况，足以证明全文索引能加快检索速度。

###### 查询扩展

查询扩展用来设法放宽所返回的全文本搜索结果的范围。考虑下面的情况。你想找出所有提到anvils的注释。只有一个注释包含anvils，但你还想找出可能与你搜索有关的其他行，即使它们不包含词anvils。

在使用查询扩展时，MySQL对数据和索引进行两边扫描来完成搜索：

* 首先，进行一个基本的全文本搜索，找出与搜索条件匹配的所有行；
* 其次，MySQL检查这些匹配行并选择所有有用的词；
* 而其次，MySQL再进行全文本搜索，这次不仅使用原来的条件，而且还使用所有有用的词。

```sql
select note_text from productnotes where match(note_text) against('anvils');
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                                                                |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Multiple customer returns, anvils failing to drop fast enough or falling backwards on purchaser. Recommend that customer considers using heavier anvils. |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

select note_text from productnotes
where match(note_text) against('anvils' with query expansion);
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                                                                |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
| Multiple customer returns, anvils failing to drop fast enough or falling backwards on purchaser. Recommend that customer considers using heavier anvils. |
| Customer complaint:
Sticks not individually wrapped, too easy to mistakenly detonate all at once.
Recommend individual wrapping.                         |
| Customer complaint:
Not heavy enough to generate flying stars around head of victim. If being purchased for dropping, recommend ANV02 or ANV03 instead.  |
| Please note that no returns will be accepted if safe opened using explosives.                                                                            |
| Customer complaint: rabbit has been able to detect trap, food apparently less effective now.                                                             |
| Customer complaint:
Circular hole in safe floor can apparently be easily cut with handsaw.                                                               |
| Matches not included, recommend purchase of matches or detonator (item DTNTR).                                                                           |
+----------------------------------------------------------------------------------------------------------------------------------------------------------+
7 rows in set (0.00 sec)
```

这次返回了7行。第一行包含词anvils，因此等级最高。第二行与anvils无关，但因为它包含第一行中的两个词（ customer和recommend），所以也被检索出来。第3行也包含这两个相同的词，但它们在文本中的位置更靠后且分开得更远，因此也包含这一行，但等级为第三。第三行确实也没有涉及anvils（按它们的产品名）。  

###### 布尔文本搜索

MySQL支持全文本搜索的另外一种形式，称为布尔方式(boolean mode)。以布尔方式，可以提供关于如下内容的细节：

* 要匹配的词；
* 要排斥的词(如果某行包含这个词，则不返回该行，即使它包含其他指定的词也是如此)；
* 排列提示(指定某些词比其他词重要，更要的的词等级更高)；
* 表达式分组；
* 另外一些内容。

`Mark： 布尔方式不同于迄今为止使用的全文本搜索语句的地方在于，即使没有定义fulltext索引，也可以使用它。但这是一种非常缓慢的操作(其性能将随着数据量的增加而降低)`

```sql
select note_text from productnotes
where match(note_text) against('heavy' in boolean mode);

+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                                                               |
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| Item is extremely heavy. Designed for dropping, not recommended for use with slings, ropes, pulleys, or tightropes.                                     |
| Customer complaint:
Not heavy enough to generate flying stars around head of victim. If being purchased for dropping, recommend ANV02 or ANV03 instead. |
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

此全文本搜索检索包含词heavy的所有行。其中使用了关键字in boolean mode，但实际上没有指定布尔操作符，因此其结果与没有指定布尔方式的结果相同。

`Mark：虽然这个例子的结果与没有in boolean mode的相同，但其行为有一个重要的差别`

为了匹配包含heavy但不包含以rope开始的词的行

```sql
select note_text from productnotest
where match(note_text) against('heavy -rope*' in boolean mode);

+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| note_text                                                                                                                                               |
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| Customer complaint:
Not heavy enough to generate flying stars around head of victim. If being purchased for dropping, recommend ANV02 or ANV03 instead. |
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

使用-rope\*明确地只是MySQL排除包含rope\*的行。

全文本布尔操作符

| 布尔操作符 | 说明                                    |
| ----- | ------------------------------------- |
| +     | 包含，词必须存在                              |
| -     | 排除，词必须不存在                             |
| >     | 包含，而且增加等级值                            |
| <     | 包含，且减少等级值                             |
| ()    | 把词组成子表达式(允许这些子表达式作为一个组被包含、排除、排列等)     |
| ~     | 取消一个词的排序值                             |
| *     | 词尾的通配符                                |
| ""    | 定义一个短语(与单个词的列表不一样，它匹配整个短语以便包含或排除这个短语) |

```sql
select note_text from productnotes
where match(note_text) against('+rabbit +bait' in boolean mode);
## 这个搜索匹配包含词rabbit和bait的行

select note_text from productnotes
where match(note_text) against('rabbit bait' in boolean mode);
## 没有指定操作符，这个搜索匹配包含rabbit和bait中的至少一个词的行

select note_text from productnotes
where match(note_text) against('"rabbit bait"' in boolean mode);
## 匹配短语rabbit abit而不是匹配两个词

select note_text from productnotes
where match(note_text) against('>rabbit <carrot' in boolean mode);
## 匹配rabbit和carrot，增加前者等级，降低后者等级

select note_text from productnotes
where match(note_text) against('+safe +(<combination)' in boolean mode);
## 匹配词safe和combination，降低后者的等级。
```

`Mark：在布尔方式中，不按等级值降序排序返回的行`

关于全文本搜搜的某些重要说明：

* 在索引全文本数据时，短词被忽略且从索引中排除。短词定义为那些具有3个或3个以下字符的词(如果需要，这个数值可以更改)。
* MySQL带有一个内建的非用词(stopword)列表，这些词在索引全文本数据时总是被忽略。如果需要，可以覆盖这个列表。
* 许多词出现的频率很高，搜索它们没有用处(返回太多结果)。因此，MySQL规定了一条50%规则，如果一个词出现在50%以上的行中，则将它作为一个非用词忽略。50%规则不用于In Boolean Mode。
* 如果表中的行数少于3行，则全文本搜索不返回结果(因为每个词或者不出现，或者至少出现在50%的行中)。
* 忽略词中的单引号。例如，don't 索引为 dont。
* 不具有词分隔符的语言不能恰当地返回全文本搜索结果。
* 如前所述，仅在MyISAM引擎中支持全文本搜索。

#### 插入数据

提高整体性能：

数据库经常被多个客户访问，对处理什么请求以及用什么次序处理进行管理是MySQL的任务。Insert操作可能很耗时(特别是有很多索引需要更新时)，而且它可能降低等待处理的select语句的性能。如果数据检索是最重要的，则可以通过在insert和into之间添加关键字low_priority，指示MySQL降低Insert语句的优先级，insert low_priority into，此方法也适用于update和delete。

单条Insert语句处理多个插入比使用多条Insert语句快。

#### 更新和删除数据

如果用Update语句更新多行时，并且在更新这些行中的一行或多行时出现一个错误，则整个Update操作被取消(错误发生前更新的所有行被恢复到它们原来的值)。为即使是发生错误，也继续进行更新，可以使用ignore关键字，update ignore 表名 set 列名=修改值 where 列名=条件

delete 删除表中所有记录的性能不如truncate，是因为truncate是直接将表删除然后重新创建一张空表，而delete是逐行删除记录。

#### 视图

视图本身不包含数据，查询视图返回的数据都是从其他表汇总检索出来的，如果表中的数据进行过修改，视图将返回改变过的数据。

**性能问题**：因为视图不包含数据，所以每次查询视图时，都必须处理查询所需要的任一个检索。如果用了多个联结和过滤创建了复杂的视图或者嵌套了视图，可能会发现性能下降得很厉害。

视图的规则和限制：

* 与表一样，必须唯一命名。
* 对于可以创建的视图数目没有限制。
* 为了创建视图，必须具有足够的访问权限。这些限制通常由DBA授予。
* 视图可以嵌套，即可以利用从其他视图中检索数据的查询来构造一个视图。
* order by可以用在视图中，但如果从该视图检索数据select中页含有order by，那么该视图中的order by将被覆盖。
* 视图不能索引，也不能有关联的触发器或默认值。
* 视图可以和表一起使用。

更新视图，即更新其基表，对视图的增删改都是对其基表进行增删改。但是，并非所有的视图都是可更新的。基本上可以说，如果MySQL不能正确地确定被更新的基数据，则不允许更新(包括插入和更新)。这实际上意味着，如果视图定义中有以下操作，则不能进行视图的更新：

* 分组(使用group by 和 having)
* 联结
* 子查询
* 并
* 聚集函数
* distinct
* 导出(计算)列

#### 存储过程

变量 (Variable) 内存中一个特定的位置，用来临时存储数据。

```sql
delimiter //
create procedure productpricing(
    OUT pl decimal(8,2),
    OUT ph decimal(8,2),
    OUT pa decimal(8,2)
)
begin
    select min(prod_price) into pl from products;
    select max(prod_price) into ph from products;
    select avg(prod_price) into pa from products;
end //
delimiter ;
```

此存储过程接受3个参数：pl存储产品最低价格，ph存储产品最高价格，pa存储产品平均价格。每个参数必须具有指定的类型，这里使用十进制。关键字 **OUT** 指出相应的参数用来从存储过程传出一个值(返回给调用者)。MySQL支持 **IN** (传递给存储过程)、**OUT** (从存储过程传出)、**INOUT** (对存储过程传入和传出) 类型的参数。存储过程的代码位于begin和end语句内。

为调用此存储过程，必须指定3个变量名

```sql
call productpricing(@pricelow,
                    @pricehigh,
                    @priceaverage);
## 所有的变量都必须以 @ 开始
## 执行完存储过程后，就可以直接使用变量
select @pricehigh, @pricelow, @priceaverage;
+------------+-----------+---------------+
| @pricehigh | @pricelow | @priceaverage |
+------------+-----------+---------------+
|      55.00 |      2.50 |         16.13 |
+------------+-----------+---------------+
1 row in set (0.00 sec)
```

下面再举一个例子

```sql
delimiter //
create procedure ordertotal(
    IN onumber int,
    OUT ototal decimal(8,2)
)
begin
    select sum(item_price*quantity)
    from orderitems
    where order_num = onumber
    into ototal;
end //
delimiter ;

call ordertotal(20005, @total);

select @ototal;
+--------+
| @total |
+--------+
| 149.87 |
+--------+
1 row in set (0.00 sec)
```

一个较为复杂的存储过程

```sql
delimiter //
-- Name: ordertotal
-- Parameters: onumber = order number
--                taxable = 0 if not taxable, 1 if taxable
--                ototal = order total variable
create procedure ordertotal(
    IN onumber INT,
    IN taxable BOOLEAN,
    OUT ototal DECIMAL(8,2)
) comment 'Obtain order total, optionally adding tax'
begin


    -- declare variable for total
    declare total decimal(8,2);
    -- declare tax percentage
    declare taxrate int default 6;

    -- get the order total
    select sum(item_price*quantity)
    from orderitems
    where order_num = onumber
    into total;

    -- is this taxable?
    if taxable then
        -- yes, so add taxrate to the total
        select total+(total/100*taxrate) into total;
    end if;

    -- and finally, save to out variable
    select total into ototal;
end //
delimiter ;

call ordertotal(20005, 0, @total);
Query OK, 1 row affected (0.00 sec)

mysql> select @total;
+--------+
| @total |
+--------+
| 149.87 |
+--------+
1 row in set (0.00 sec)

call ordertotal(20005, 1, @total);
Query OK, 1 row affected (0.00 sec)

mysql> select @total;
+--------+
| @total |
+--------+
| 158.86 |
+--------+
1 row in set (0.00 sec)
```

#### 游标

使用游标涉及几个明确的步骤。

* 在能够使用游标前，必须声明(定义)它。这个过程实际上没有检索数据，它只是定义要使用的select语句。
* 一旦声明后，必须打开游标以供使用。这个过程用前面定义的select语句把数据实际检索出来。
* 对于填有数据的游标，根据需要取出(检索)各行。
* 在结束游标使用时，必须关闭游标。

在声明游标后，可根据需要频繁地打开和关闭游标。在游标打开后，可根据需要频繁地执行取操作。

创建游标：用declare语句创建。declare命名游标，并定义相应的select语句，根据需要带where和其他子句。

```sql
create procedure processorders()
begin
    declare ordernumbers cursor
    for
    select order_num from orders;
end
```

这个存储过程用declare语句定义和命名游标，这里为ordernumbers。存储过程处理完成后，游标就消失（因为它局限于存储过程）。在定义游标之后，可以打开它。

打开游标：open cursor

```sql
open ordernumbers;
```

在处理open语句时执行查询，存储检索出的数据以供浏览和滚动。

游标使用完之后，使用如下语句关闭游标：

```sql
close cursor;
close ordernumbers;
```

close释放游标使用的所有内部内存和资源，因此在每个游标不再需要时都应该关闭。

在一个游标关闭后，如果没有重新打开，则不能使用它。但是，使用声明过的游标不需要再次声明，用open语句打开就可以了。

`Mark：如果你不明确关闭游标，MySQL将会在到达END语句时自动关闭它。`

修改前面一个例子：

```sql
create procedure processorders()
begin
    -- declare the cursor
    declare ordernumbers cursor
    for
    select order_num from orders;

    -- open the cursor
    open ordernumbers;

    -- close the cursor
    close ordernumbers;
end
```

使用游标数据：

```sql
create procedure processorders()
begin
    declare o int;
    declare ordernumbers cursor
    for
    select order_num from orders;
    open ordernumbers;
    fetch ordernumbers into o;
    close ordernumbers;
end
```

使用fetch用来检索当前行的order_num列（将自动从第一行开始）到一个名为o的局部声明变量中。对检索出的数据不做任何处理。

```sql
delimiter //
create procedure processorders()
begin
    declare done boolean default 0;
    declare o int;
    declare ordernumbers cursor
    for
    select order_num from orders;
    declare continue handler for sqlstate '02000' set done = 1;
    open ordernumbers;
    repeat
        fetch ordernumbers into o;
    until done end repeat;
    close ordernumbers;
end //
delimiter ;
```

为了进一步把这些内容组织起来，这里给出更详细的版本

```sql
delimiter //
create procedure processorders()
begin
    declare done boolean default 0;
    declare o int;
    declare t decimal(8,2);

    declare ordernumbers cursor
    for
    select order_num from orders;
    declare continue handler for sqlstate '02000' set done = 1;

    create table if not exists ordertotals(
        order_num int,
        total decimal(8,2)
    );

    open ordernumbers;
    repeat
        fetch ordernumbers into o;
        call ordertotal(o, 1, t);
        insert into ordertotals(order_num, total)
        values(o, t);
        until done end repeat;
    close ordernumbers;
end //
delimiter ;
```

这个例子中，增加了一个变量t，存储每个订单的合计。此外，还创建了一个新表，名为ordertotals。这个表将保存存储过程生成的结果。fetch像以前一样取每个order_num，然后用call执行另一个存储过程来计算每个订单的带税的合计。最后用insert保存每个订单的订单号和合计。

此存储过程不返回数据，但它能够创建和填充另一个表，可以用一条简单的select语句查看该表。

#### 触发器

###### 创建触发器

* 唯一的触发器名
* 触发器关联的表
* 触发器应该响应的活动(delete、insert或update)
* 触发器何时执行

```sql
create trigger newproduct after insert on products
for each row select 'product added';
```

在每次insert products表后显示一次'product added'；为什么是每一行都要显示，是因为使用了for each row。

`Mark：只有表才支持触发器，视图不支持（临时表也不支持）`

单一的触发器不能与多个事件或多个表关联，所以，如果你需要一个对insert和update操作执行的触发器，则应该定义两个触发器。

`Mark：如果before触发器失败，则MySQL将不执行请求的操作。此外，如果before触发器或语句本身失败，MySQL将不执行after触发器（如果有的话）`

###### 删除触发器

```sql
drop trigger newproduct;
```

触发器不能更新或覆盖。为了修改一个触发器，必须先删除它，然后再重新创建。

###### 使用触发器

######## Insert触发器

insert触发器在执行insert语句之前或之后执行。需要知道以下几点：

* 在insert触发器代码内，可引用一个名为new的虚拟表，访问被插入的行；
* 在before insert触发器中，new中的值也可以被更新（允许更改被插入的值）；
* 对于auto_increment列，new在insert执行之前包含0，在inset执行之后包含新的自动生成值。

```sql
create trigger neworder after insert on orders
for each row select new.order_num;
```

此代码创建一个名为neworder的触发器，按照after insert on orders 执行。触发器从new.order_num取得这个值并返回它。此触发器必须按照after insert 执行，因为在before insert语句执行之前，新order_num还没有生成。对于orders的每次插入使用这个触发器总是返回新的订单号。

`Mark： Before or After？ 通常，将before用于数据验证和净化（目的是保证插入表中的数据确实是需要的数据）`

######## delete触发器

delete触发器在delete语句执行之前或之后执行，需要知道以下两点：

* 在delete触发器代码内，你可以引用一个名为old的虚拟表，访问被删除的行；
* old中的值全都是只读的，不能更新。

```sql
delimiter //
create trigger deleteorder before delete on orders
for each row
begin
    insert into archive_orders(order_num, order_date, cust_id)
    values(old.order_num, old.order_date, old.cust_id);
end //
delimiter ;
```

在任意订单被删除前执行此触发器。使用一条insert语句将old中的值保存到一个名为archive_orders的存档表中。

使用before delete触发器的有点（相对于after delete触发器来说），如果由于某种原因，订单不能存档，delete本身将被放弃。

`Mark：触发器deleteorder使用begin和end语句标记触发体。在这个例子中不是必须的，不过也没有害处。使用begin end块的好处是触发器能容纳多条SQL语句`

######## update触发器

update触发器在update语句执行之前或之后执行。需要知道以下几点：

* 在update触发器代码中，可以引用一个名为old的虚拟表访问以前（update语句前）的值，引用一个名为new的虚拟表访问新更新的值；
* 在before update触发器中，new中的值可能也被更新（允许更改将要用于update语句中的值）；
* old中的值全都是只读，不能更新。

```sql
create trigger updateoder before update on orders
for each row set new.vend_state = Upper(new.vend_state);
```

在每次更新之前都对字段进行大写处理

#### 管理事务

事务处理是一种机制，用来管理必须成批执行的MySQL操作，以保证数据不包含不完整的操作结果。利用事务处理，可以保证一组操作不会中途停止，它们或者作为整体执行，或者完全不执行。如果没有错误发生，整租语句提交给数据库表。如果发生错误，则进行回退以恢复数据库到某个已知且安全的状态。

在使用事务和事务处理时，有几个需要知道术语：

* 事务（transaction）指一组SQL语句；
* 回退（rollback）撤销指定SQL语句的过程；
* 提交（commit）将未存储的SQL语句结果写入数据库表；
* 保留点（savepoint）事务处理中设置的临时占位符（placeholder），你可以对它发布回退（与回退整个事务处理不同）。

```sql
## 标记一个事务的开始
start transaction;

## 回退
rollback；

## 事务回退仅支持DML语句，即 insert update delete
## 但是DDL语句，是立即生效机制，所以不会回退

## 提交
commit;

## 保留点
savepoint delete1;
## 回退到保留点
rollback delete1;
```

简单的rollback和commit语句就可以写入或撤销整个事务处理。但是，只是对简单的事务处理才这样做，更复杂的事务可能需要部分提交或回退。

为了支持回退部分事务，必须在事务处理块中合适的位置放置占位符。这样，如果要回退，可以回退到某个占位符。这些占位符称为保留点。

`Mark：保留点越多越好？可以在MySQL代码中设置任意多的保留点，越多越好。因为保留点越多，你就越能按自己的意愿灵活进行回退`

`Mark：释放保留点   保留点在事务处理完成（执行一条rollback或commit）后自动释放。也可以使用release savepoint明确的释放保留点`

正常情况下，MySQL默认设置为自动提交模式，可以通过设置autocommit值来动态修改是否自动提交。1：开启自动提交，0：关闭自动提交。

#### 字符集

字符集的继承关系：

字段继承--->表继承--->库

简而言之，创建库的时候默认不设置字符集，则使用服务器配置文件中的默认值

创建表的时候不设置字符集，则使用数据库配置的字符集

创建表中的字段时，不单独设置字符集，则使用表的字符集

#### 权限

| 权限                      | 说明                                                                  |
| ----------------------- | ------------------------------------------------------------------- |
| all                     | 除grant option外的所有权限                                                 |
| alter                   | 使用alter table                                                       |
| alter routine           | 使用alter procedure和drop procedure                                    |
| create                  | 使用create table                                                      |
| create routine          | 使用create procedure                                                  |
| create temporary tables | 使用create temporary table                                            |
| create user             | 使用create user、drop user、rename user和revoke all privileges           |
| create view             | 使用create view                                                       |
| delete                  | 使用delete                                                            |
| drop                    | 使用drop table                                                        |
| execute                 | 使用call和存储过程                                                         |
| file                    | 使用select into outfile和load data infile                              |
| grant option            | 使用grant和revoke                                                      |
| index                   | 使用create index和drop index                                           |
| insert                  | 使用insert                                                            |
| lock tables             | 使用lock tables                                                       |
| process                 | 使用 show full processlist                                            |
| reload                  | 使用flush                                                             |
| replication client      | 服务位置的访问                                                             |
| replication slave       | 由复制从属使用                                                             |
| select                  | 使用select                                                            |
| show databases          | 使用show databases                                                    |
| show view               | 使用show create view                                                  |
| shutdown                | 使用mysqladmin shutdown(用来关闭MySQL)                                    |
| super                   | 使用change master、kill、logs、purge、master和set global。还允许mysqladmin调试登录 |
| update                  | 使用update                                                            |
| usage                   | 无访问权限                                                               |

`未来的授权：在使用grant和revoke时，用户账号必须存在，但对所涉及的对象没有这个要求。这允许管理在创建数据库和表之前设计和实现安全措施。这样做的副作用是，当某个数据库或表被删除时(用Drop语句)，相关的访问权限依然存在。而且，如果将来重建这些库表，这些访问权限依然生效`

#### 数据库维护

`首先刷新未写数据：为了保证所有数据被写到磁盘（包括索引数据），可能需要在进行备份前使用flush tables语句`

```sql
## analyze用来检查表键是否正确
analyze table orders;

## check table用来针对许多问题对表进行检查
check table orders, orderitems;

## 如果MyISAM表访问产生不正确和不一致的结果，需要用repair table来修复相应的表
repair table orders;

## 如果从一个表中删除大量数据，应该使用optimize table来回收所用的空间，从而优化表的性能
optimize table orders;
```

#### 数据类型整理

字符串

| 数据类型       | 说明                                                            |
| ---------- | ------------------------------------------------------------- |
| char       | 1~255个字符定长字符串。长度必须在创建时指定，否则MySQL假定为char(1)                    |
| enum       | 接受最多64k个字符串组成的一个预定义结合的某个字符串                                   |
| longtext   | 与text相同，但最大长度为4GB                                             |
| mediumtext | 与text相同，但最大长度为16K                                             |
| set        | 接受最多64个字符串组成的一个预定义集合的零个或多个字符串                                 |
| text       | 最大长度为64K的变长文本                                                 |
| tinytext   | 与text相同，但最大长度为255字节                                           |
| varchar    | 长度可变，最多不超过255字节。如果在创建时指定为varchar(n)，则可存储0~n个字符的变长字符串（其中n≤255） |

数值

| 数据类型         | 说明                                                                                  |
| ------------ | ----------------------------------------------------------------------------------- |
| bit          | 位字段，1~64位                                                                           |
| bigint       | 整数值，支持-8223372036854775808~9223372036854775807，如果是unsigned，为0~1844674407370955615的数 |
| boolean或bool | 布尔标志，或者为0或者为1，主要用于开/关(on/off)标志                                                     |
| decimal或dec  | 精度可变的浮点值                                                                            |
| double       | 双精度浮点值                                                                              |
| float        | 单精度浮点值                                                                              |
| int或integer  | 整数值，支持-2147483648~2147483647，如果是unsigned，为0~4294967295                              |
| mediumint    | 整数值，支持-83388608~8388607，如果是unsigned，为0~16777215                                     |
| real         | 4字节的浮点值                                                                             |
| smallint     | 整数值，支持-32768~32767，如果是unsigned，为0~65535                                             |
| tinyint      | 整数值，支持-128~127，如果是unsigned，为0~255                                                   |

日期和时间

| 数据类型      | 说明                                                  |
| --------- | --------------------------------------------------- |
| date      | 表示1000-01-01~9999-12-31的日期，格式为YYYY-MM-DD            |
| datetime  | date和time的组合                                        |
| timestamp | 功能和datetime相同（但范围较小）                                |
| time      | 格式为HH：MM：SS                                         |
| year      | 用2位数字表示，范围是70(1970年)~69(2069年)，用4位数字表示范围是1901~2155年 |

二进制

| 数据类型       | 说明             |
| ---------- | -------------- |
| blob       | Blob最大长度为64KB  |
| mediumblob | Blob最大长度为16MB  |
| longblob   | Blob最大长度为4GB   |
| tinyblob   | Blob最大长度为255字节 |

------
