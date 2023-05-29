---
layout: "@layouts/ArticleLayout.astro"
title: Jmeter学习笔记
description: Jmeter学习笔记
date: 2023-02-21 17:41:54
tags:
  - Jmeter
  - LearnNote
  - 笔记
---

## 初识Jmeter

Apache JMeter是Apache组织开发的基于Java的压力测试工具。用于对软件做压力测试，它最初被设计用于Web应用测试，但后来扩展到其他测试领域。 它可以用于测试静态和动态资源，例如静态文件、Java [小服务程序](https://baike.baidu.com/item/小服务程序/4148836)、CGI 脚本、Java 对象、数据库、[FTP](https://baike.baidu.com/item/FTP/13839) 服务器， 等等。JMeter 可以用于对服务器、网络或对象模拟巨大的负载，来自不同压力类别下测试它们的强度和分析整体性能。另外，JMeter能够对应用程序做功能/[回归测试](https://baike.baidu.com/item/回归测试/1925732)，通过创建带有断言的脚本来验证你的程序返回了你期望的结果。为了最大限度的灵活性，JMeter允许[使用正则表达式](https://baike.baidu.com/item/使用正则表达式/6555484)创建断言。

Apache jmeter 可以用于对静态的和动态的资源（文件，Servlet，Perl脚本，java 对象，数据库和查询，[FTP服务器](https://baike.baidu.com/item/FTP服务器)等等）的性能进行测试。它可以用于对服务器、网络或对象模拟繁重的负载来测试它们的强度或分析不同压力类型下的整体性能。你可以使用它做性能的图形分析或在大并发[负载测试](https://baike.baidu.com/item/负载测试/10921210)你的服务器/脚本/对象。

## 组件讲解

> Jmeter的组件以单节点树的形式展示，一个组件依附在另一个组件下面或者与另一个组件同级

### Test Plan

虽然名字是测试计划，但是跟测试活动中的“测试计划”是两回事。这里叫测试计划，更多的是从管理的角度上出发，一个计划管理N个线程组，N个线程组执行不同的请求……

在这一层级，可以添加一些自定义的变量，甚至可以添加一些Jar包到classpath中。

### Thread Group

线程组是测试计划的起点元素，其他的元素都是添加在线程组之下的

线程组可以配置的参数较多，结合文档+目前的理解情况，

*  Action to be taken after a Sampler error：在取样器失败后应采取什么操作
   * Continue：默认选项，继续执行
   * Start Next Thread Loop：启动下一次线程循环。当线程组内存在多个线程时，其中一个失败后，将重跑所有线程
   * Stop Thread：停止当前线程
   * Stop Test：停止测试计划，正在执行中的线程，需要等待其结束
   * Stop Test Now：停止测试计划，强制停掉所有正在执行的线程
* Number of Threads (users)：一个线程组应包含多少个线程
* Ramp-up period (seconds)：启动完线程组内所有的线程需要多少秒。比如要启动10个线程，Ramp-up period设置为10，即间隔1秒启动一个线程，有点类似集结点，但不全是。
* Loop Count：循环次数，循环整个线程组，支持永久循环
* Same user on each iteration：在每个线程组迭代上，控制模拟同一个用户运行多次迭代，还是不同用户运行一次迭代
* Delay Thread creation until needed：延迟线程启动直到需要为止
* Specify Thread lifetime(确定线程的生命周期)
	* Duration (seconds)：一个线程执行的时长
	* Startup delay (seconds)：一个线程启动的延时

## Controllers

> JMeter has two types of Controllers: Samplers and Logical Controllers. These drive the processing of a test.

### Samplers

> Samplers tell JMeter to send requests to a server and wait for a response. They are processed in the order they appear in the tree. Controllers can be used to modify the number of repetitions of a sampler.

* FTP Request
* HTTP Request (can be used for SOAP or REST Webservice also)
* JDBC Request
* Java object request
* JMS request
* JUnit Test request
* LDAP Request
* Mail request
* OS Process request
* TCP request





### Logical Controllers

> Logical Controllers let you customize the logic that JMeter uses to decide when to send requests.



## Listener 监听器



## Timers 定时器



## Assertions 断言



## Configuration Elements 配置元素



## Pre-Processor Elements 预置处理器



## Post-Processor Elements 后置处理器



## Scoping Rules 规则作用域



## Properties and Variables 属性与变量



## Using Variable to parameterise tests 使用变量参数化测试



# 执行顺序

1. Configuration elements 配置元素
2. Pre-Processors 预处理器
3. Timers 定时器
4. Sampler 取样器/采集器
5. Post-Processors (unless SampleResult is **null**) 后处理器（除非 SampleResult 为**null**）
6. Assertions (unless SampleResult is **null**) 断言（除非 SampleResult 为**null**）
7. Listeners (unless SampleResult is **null**) 侦听器（除非 SampleResult 为**null**）

