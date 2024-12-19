---
layout: "@layouts/ArticleLayout.astro"
title: selenium中执行JavaScript的过程解析
description: selenium中执行JavaScript的过程解析
date: 2024-12-19 21:13:13
tags: 
    - selenium
---

> 本文章摘抄至本人CSDN文章

# 使用方式
```python
# 同步执行js代码
driver.execute_script(script, *args)
"""
Synchronously Executes JavaScript in the current window/frame.

:Args:
  - script: The JavaScript to execute.
  - \\*args: Any applicable arguments for your JavaScript.
:Usage:
   ::
   driver.execute_script('return document.title;')
"""

# 异步执行js代码
driver.execute_async_script(self, script: str, *args)
"""
Asynchronously Executes JavaScript in the current window/frame.
:Args:
  - script: The JavaScript to execute.
  - \\*args: Any applicable arguments for your JavaScript.
:Usage:
  ::
  script = "var callback = arguments[arguments.length - 1]; " \\
           "window.setTimeout(function(){ callback('timeout') }, 3000);"
  driver.execute_async_script(script)
"""
```

上面两段代码注释，均来自于python的selenium包里面

# 同步执行js
所谓的同步，其实是selenium将需要执行的js代码发给浏览器之后，需要等待浏览器立即响应并给出返回值。所以，我们要执行的js代码必须能立即得到返回值，否则将会出现超时。尽管是同步执行，但是也有一个超时检测机制，默认30秒。
示例中的代码执行的是 return document.title
单纯的理解document.title还好，但是为什么要加上一个return？
return不应该是一个函数给调用方返回结果时，才需要的吗？
其实，我们的js代码就是被封装进了一个匿名的js函数中，当只需要通过js点击元素、滚动页面等操作时，是不需要返回值得，所以不需要携带return。不过，在我们需要获取某个元素的某个属性时，selenium自带的方法解决不了，可以从js很方便的获取时，就需要用return将这个属性值给拿到。大致可以抽象理解为如下：
```javascript
function anonymous(script) {
	/* 伪代码
	解析传入的待执行script
	执行script
	如果有return：
		将结果返回
	如果没有return：
		直接结束
	*/
}
/*
我们在Python中执行的driver.execute_script("return document.title")
就是把这段代码放到了anonymous函数内部，让我们要执行的代码封装在一个js函数里面
再通过浏览器去解析执行
*/
```
# 异步执行js
介绍完同步执行，异步执行就稍微简单一些
异步执行也就是将我们的代码放在了函数内部，不过由于我们请求的是异步，所以不能立即返回结果。在js中，不能立即得到返回值的调用，都需要一个回调函数来获取结果。
还是用示例代码说明：
```python
script = "var callback = arguments[arguments.length - 1]; " \\
         "window.setTimeout(function(){ callback('timeout') }, 3000);"
driver.execute_async_script(script)
"""
重点看js代码使如何书写的。
var callback = arguments[arguments.length - 1];
arguments.length：指的是js函数的的入参个数；js函数中有一个特殊的变量是arguments，记录了函数到底有多少个入参，用一个数组存放
arguments[arugments.length - 1]：js函数的最后一个入参
整句话就是说，把函数的最后一个入参赋值给了变量callback，但是要注意js是弱类型语言，callback可以是变量也可以是函数、类。在这里不管callback是什么，它要做的就是把结果返回出来

window.setTimeout(function(){ callback('timeout') }, 3000);
setTimeout是启动一个定时后台任务，接受2个参数，第一个是要执行的函数，第二个是延迟多少毫秒后执行
这里的含义是，在3秒之后执行一个函数，函数执行的内容是通过callback将'timeout'返回出去
所以driver.execute_async_script(script)得到一个返回值'timeout'
"""
```
实际上js实现的回调是通过promise做的，关于promise这里有详细的介绍[Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)

如果只是单纯调用，只需要知道异步执行的时候，必须要指定一个callback函数来回调结果。


后面会再出一篇文章，是本人用selenium完成接口测试的案例，是一个剑走偏锋的操作，只做知识普及，不建议落地执行。