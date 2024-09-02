---
layout: "@layouts/ArticleLayout.astro"
title: Python2中Unicode字符串与字节字符串
description: Unicode字符串与字节字符串
date: 2024-09-02 17:20:32
tags: 
    - featured
---

> 背景: 一直以来都是在用Python2作为自动化脚本的编程语言，所以一直存在编码问题，只是不痛不痒没到彻底根除的地步
> 此篇文章用于总结，在Python2中如何处理Unicode与字节型字符串

脚本书写过程中，容易出现的Unicode与ASCII编码的问题

因此，在确定字符的时候，就遇到了ASCII码的普通字母与Unicode的中文以及符号的“碰撞”。

先说过程中遇到的问题。

1.普通形式的中文，无法正确编码，会如下类似的错误
```python
UnicodeDecodeError: 'ascii' codec can't decode byte 0xef in position 297: ordinal not in range(128)
```
2.在校验返回值的时候，无法正确的做 等于、不等于 的比较
3.Unicode的符号、中文无法直接当做参数调用API
4.在抛出自定义异常时，定义的错误信息中，ASCII与Unicode无法直接拼接成一个“字符串”

接下来仔细解释上面提到的问题：

### 什么是普通形式的中文？普通形式的中文，这个说法仅存在于Python2中，这种的形式的中文如下：
```python
_str = "我是中文"
```
这里的_str变量，通过 print 或者 写入日志，都是没问题的，都能正常转码(Python2中，前提是文件顶部，需要标注 `# -*- coding: utf-8 -*-`)。这样文件中的所有编码都是按照utf8进行存储。

既然如此，什么样的情况会出现上述异常呢？就是下面这种情况：
```python
_str1 = "abcde"
_str2 = u"我是中文"
print(_str1 + _str2)
```
出现这种拼接或者混合ASCII与Unicode使用的场景，是因为在校验特殊字符的时候，是随机生成的。生成随机字符串，就需要一个“池子”来预装一些字符串，这其中就包含了英文字母、英文标点符号、英文特殊字符、中文、中文标点符号等等。

当这些ASCII与Unicode在一个“池子”做随机化拼接的时候，就必须统一编码形式，就会出现Unicode无法正确的编码为ASCII码(这个编码顺序是Python2固定的)。所以处理的办法，是统一编码形式，即统一成Unicode。当然，这里统一成Unicode是因为支持性较好，且Python3中字符串也只有Unicode这一种形式，这样做的好处就是向Python3看齐。

额外说一点，肯定有人想问，为什么不直接将中文前面的 u 去掉？实际上，去掉 u ，的确不会影响生成的随机字符串，但在后续的API调用过程中，将会报错，错误也是编码的问题。原因是，Python2中不带 u 的字符串，实际上都是字节码形式的字符串，为了证明它的确是字节码，在交互式终端中，可以通过repr("中文")，看控制台给出的回显是否是 '\x1574\x1515'形式。这种以 \x 开头的就表示字节码，为什么中文就非得是字节码呢？是因为中文无法通过ASCII码来定义，因为ASCII只有128的长度，但中文远远超过这个编码长度，所以会出现编码问题，在内部处理过程中对中文，都是按照字节码来处理。API调用过程中，框架的逻辑里会将参数转换成JSON格式发送出去，在序列化成JSON时，因为存在字节码(中文)，所以JSON会将其转换成Unicode(因为JSON处理不了Python定义的字节码)，这时候由于JSON无法确定一个中文到底是占几个位，而出现编码问题。

所以，最好的解决办法就是先统一内部的编码为Unicode，解决掉API调用会失败的问题。至于字符串拼接之后的print输出、日志的写入将采用另外的方法来兼容。

### 校验返回值的时候，无法正确的做 等于、不等于 的比较
这是因为 字节码 形式的字符串与 Unicode 形式的字符串不能简单的用 = !=来做比较，毕竟编码形式不一致，无法比较。还是统一成Unicode之后来做校验。

这里的情况，只会出现在API接口返回的数据中存在Unicode字符(即有中文)的情况。类似场景如下：
```python
param = {
  "Name": "我是中文"
}
response = {
  "RequestId": "xxx-xxxxx-xxxxx-xxxxx"
  "Name": "\u6211\u662f\u4e2d\u6587"
}

if param["Name"] == response["Name"]:
    print True
else:
    print False
```
这里肯定是输出False，可以自行在Python2的交互式终端下进行模拟。比如这里给出模拟结论：
```python
PyDev console: starting.
Python 2.7.18 (v2.7.18:8d21aa21f2, Apr 20 2020, 13:25:05) [MSC v.1500 64 bit (AMD64)] on win32
a = "\u6211\u662f\u4e2d\u6587"
b = "我是中文"
a == b
False
```
出现这种情况的原因是，API调用后返回的结果中包含了中文，但结果是按照JSON存储的，在将结果反序列化成Python字典时，会对中文做Unicode编码。即将“我是中文”编码成"\u6211\u662f\u4e2d\u6587"，但param["Name"]中的中文，是按照字节码存储的，直接做比较，是不行的。

### Unicode的符号、中文无法直接当做参数调用API
这个原因，在解释第一种情况时已阐明缘由

### 在抛出自定义异常时，定义的错误信息中，ASCII与Unicode无法直接拼接成一个“字符串”
这个问题呢，还是在于字符串的拼接问题上，下面将以案例说明。
```python
# 案例一
# 自定义的报错信息中拼接了Unicode类型的字符串 和 ASCII类型的字符串

new_name = u"中文"
responsse = {
  "Name": "\u4e2d\u6587"
}
raise Exception("预期应该为 {}, 实际为 {}".format(new_name, response["Name"]))
```
这种情况，就是因为new_name是一个Unicode类型的字符串，而"预期应该为 {}, 实际为 {}"是一个字节码形式的字符串(虽然看上去都是中文)，这样拼接的时候必然会转编码格式，所以就报错了。
```python
# 案例二
new_name = "我是字节码"
responsse = {
  "Name": "\u4e2d\u6587"
}
raise Exception(u"预期应该为 {}, 实际为 {}".format(new_name, response["Name"]))
```
案例二与案例一的区别就在于，new_name不再是Unicode类型，而且Exception中的字符串也被转换成了Unicode，但依旧报错的原因是因为new_name此时已经是字节码了，拼接就必然会报错。

到此，因为字节码与Unicode编码可能出现的问题就总结完毕了。