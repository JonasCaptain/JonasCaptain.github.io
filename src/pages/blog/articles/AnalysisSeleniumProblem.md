---
layout: "@layouts/ArticleLayout.astro"
title: 分析一个关于Selenium的怪问题
description: 与selenium本身的机制有关，与浏览器记住密码功能有关
date: 2023-05-31 20:47:51
tags: 
    - featured
    - selenium
    - debug
---

> 背景：登录页面，提前在浏览器手动登录过，并且让浏览器记住了用户名和密码

```python
    driver = get_chrome_driver()
    driver.get("https://login.xxx.com")
    try:
        username = driver.find_element(By.NAME, "username")
        username.clear()
        username.send_keys("admin")
        password = driver.find_element(By.NAME, "password")
        password.clear()
        password.send_keys("admin")
    except NoSuchElementException as e:
        print(e.msg)
    time.sleep(1)
    driver.quit()
```

问题：因为保存了用户名和密码，所以在send_keys之前先clear，但是clear失败了，username Input框输入了两次admin，一次是浏览器填充进去的admin，一次是send_keys输入的admin

额外的疑问：脚本都会重新启动一个浏览器，而且这个浏览器保持足够的干净，应该不会出现记住密码的情况出现，不知道为啥会出现这个诡异的现象。

分析思路：一定是clear失败了

查找资料：
1、webDriver中关于clear的定义：
> https://w3c.github.io/webdriver/#element-clear

其中关于清理可编辑的元素时的描述如下()：
To clear a content editable element:
- If element’s innerHTML IDL attribute is an empty string do nothing and return.(元素innerHTML是空的，什么都不做，直接return)
- Run the focusing steps for element.(元素获得焦点)
- Set element’s innerHTML IDL attribute to an empty string.(将元素innerHTML清空)
- Run the unfocusing steps for the element.(释放元素焦点)

可以看到clear的第一步就是先判断元素的innertHTML是否为空。
从结果出发，肯定是判断为空了，不然也不会clear失败。但是现在要分析为什么为空？
为空的原因实际上是Input元素上被蒙了一层，这一蒙层，显示的是浏览器记住的用户名，但这用户名并没有填充在Input元素的innerHTML上，只是肉眼看上去是在Input框里。
只有当Input框获取到焦点时，浏览器才会把记住的用户名填充进去。这也就能解释，为什么出现了两次 admin。
因为send_keys获取了焦点，浏览器具有优先权，首先把记住的用户名填充进去了，后续才是send_keys传递的admin。

说到这里，已经破案了。但是这个现象的确罕见，规避的方法如下：
1、不要记住用户名和密码(推荐)
2、提前获取焦点，然后再清理
```python
    username = driver.find_element(By.NAME, "username")
    # username.click()      # 这里不建议使用click，因为click实际上点击的不是input框，而是input框上面蒙住的那一层，所以并不会让input框获取到焦点
    username.send_keys('')  # 主要为了获取焦点，有焦点浏览器就会填充记住的用户名
    username.clear()        # 有内容，Input框的innerHTML就不为空，就能清理
    username.send_keys("admin")
```
