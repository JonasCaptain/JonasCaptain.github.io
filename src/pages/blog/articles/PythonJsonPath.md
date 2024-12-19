---
layout: "@layouts/ArticleLayout.astro"
title: PythonJsonPath
description: PythonJsonPath
date: 2024-12-19 21:17:16
tags: 
    - python 
    - jsonpath
---

> 本文章摘抄至本人CSDN文章


# 1、安装方法
安装文件来源：https://pypi.org/project/jsonpath/
通过pip应该也可以安装(尚未尝试)

# 2、关于JsonPath的语法规则
JsonPath语法要点：

 - $ 表示文档的根元素 
 - @ 表示文档的当前元素 
 - .node_name 或 ['node_name'] 匹配下级节点
 -  [index] 检索数组中的元素
 -  [start : end : step] 支持数组切片语法
 - *作为通配符，匹配所有成员 
 - .. 子递归通配符，匹配成员的所有子元素 
 - (\<expr>) 使用表达式 
 - ?(<boolean expr>)进行数据筛选


|XPath|	JsonPath	|说明|
|--|--|--|
|/	|$	|文档根元素|
|.	|@|当前元素|
/|	.或[]	|匹配下级元素
..	|N/A|	匹配上级元素，JsonPath不支持此操作符
//	|..	|递归匹配所有子元素
*|	*|	通配符，匹配下级元素
@|	N/A	|匹配属性，JsonPath不支持此操作符
[]|	[]	|下标运算符，根据索引获取元素，XPath索引从1开始，JsonPath索引从0开始
|\.|[,]	|连接操作符，将多个结果拼接成数组返回，可以使用索引或别名
|N/A	|[start\: end \:step]	|数据切片操作，XPath不支持
[]	|?()	|过滤表达式
N/A	|()	|脚本表达式，使用底层脚本引擎，XPath不支持
()|	N/A	|分组，JsonPath不支持

