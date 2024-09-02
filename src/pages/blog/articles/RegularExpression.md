---
layout: "@layouts/ArticleLayout.astro"
title: Python正则表达式高阶用法
description: Python正则表达式高阶用法案例
date: 02 Sep 2024
tags:
  - Python
  - Regular Expression
---

## (?P\<name\>...)

与常规的圆括号类似，但分组所匹配到了子字符串可通过符号分组名称 name 来访问。 分组名称必须是有效的 Python 标识符，并且在 bytes 模式中它们只能包含 ASCII 范围内的字节值。 每个分组名称在一个正则表达式中只能定义一次。 一个符号分组同时也是一个编号分组，就像这个分组没有被命名过一样。
命名组合可以在三种上下文中引用。如果样式是(?P<quote>['"]).\*?(?P=quote) （也就是说，匹配单引号或者双引号括起来的字符串)：

<table>
    <tr>
    <th>引用组合"quote"的上下文</th>
    <th>引用方法</th>
    </tr>
    <tr>
        <td>在正则式自身内</td>
        <td>
            <li>(?P=quote)</li>
            <li>\1</li>
        </td>
    </tr>
    <tr>
        <td>处理匹配对象 m</td>
        <td>
            <li>m.group('quote')</li>
            <li>m.end('quote')</li>
        </td>
    </tr>
    <tr>
        <td>传递到 re.sub() 里的 repl 参数中</td>
        <td>
            <li>\g&lt;quote&gt;</li>
            <li>\g&lt;1&gt;</li>
            <li>\1</li>
        </td>
    </tr>
</table>

> 在 3.12 版本发生变更: 在 bytes 模式中，分组 name 只能包含 ASCII 范围内的字节值 (b'\x00'-b'\x7f')。

## (?P=name)

反向引用一个命名组合；它匹配前面那个叫 name 的命名组中匹配到的串同样的字串。

## (?=…)

当 … 匹配时，匹配成功，但不消耗字符串中的任何字符。这个叫做 **前视断言** （lookahead assertion）。比如， Isaac (?=Asimov) 将会匹配 'Isaac ' ，仅当其后紧跟 'Asimov' 。

## (?!…)

当 … 不匹配时，匹配成功。这个叫 **否定型前视断言** （negative lookahead assertion）。例如， Isaac (?!Asimov) 将会匹配 'Isaac ' ，仅当它后面 不是 'Asimov' 。

## (?<=…)

如果 ... 的匹配内容出现在当前位置的左侧，则匹配。这叫做 **肯定型后视断言** （positive lookbehind assertion）。 (?<=abc)def 将会在 'abcdef' 中找到一个匹配，因为后视会回退 3 个字符并检查内部表达式是否匹配。内部表达式（匹配的内容）必须是固定长度的，意思就是 abc 或 a|b 是允许的，但是 a\* 和 a{3,4} 不可以。注意，以肯定型后视断言开头的正则表达式，匹配项一般不会位于搜索字符串的开头。很可能你应该使用 search() 函数，而不是 match() 函数：

```python
import re
m = re.search('(?<=abc)def', 'abcdef')
m.group(0)
'def'
```

这个例子搜索一个跟随在连字符后的单词：

```python
m = re.search(r'(?<=-)\w+', 'spam-egg')
m.group(0)
'egg'
```

在 3.5 版本发生变更: 添加定长组合引用的支持。

## (?<!…)

如果 ... 的匹配内容没有出现在当前位置的左侧，则匹配。这个叫做 **否定型后视断言**（negative lookbehind assertion）。类似于肯定型后视断言，内部表达式（匹配的内容）必须是固定长度的。以否定型后视断言开头的正则表达式，匹配项可能位于搜索字符串的开头。

## (?(id/name)yes-pattern|no-pattern)

如果给定的 id 或 name 存在，将会尝试匹配 yes-pattern ，否则就尝试匹配 no-pattern，no-pattern 可选，也可以被忽略。比如， (<)?(\w+@\w+(?:\.\w+)+)(?(1)>|$) 是一个 email 样式匹配，将匹配 '<user@host.com>' 或 'user@host.com' ，但不会匹配 '<user@host.com' ，也不会匹配 'user@host.com>'。

> 在 3.12 版本发生变更: 分组 id 只能包含 ASCII 数码。 在 bytes 模式中，分组 name 只能包含 ASCII 范围内的字节值 (b'\x00'-b'\x7f')。

