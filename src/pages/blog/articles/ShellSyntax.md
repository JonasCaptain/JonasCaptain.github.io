---
layout: "@layouts/ArticleLayout.astro"
title: Shell脚本语法整理
description: Shell脚本语法整理
date: 2024-12-19 21:10:19
tags: 
    - shell 
    - syntax
---

> 本文章摘抄至本人CSDN文章

本文章仅用于知识归纳整理
# 知识点补充
```shell
#!/bin/bash
# 这是shell脚本的标准定义，表示使用bashell来执行脚本
echo "Hello World"
# echo用于打印文本内容到控制台
# shell脚本的执行方式
# 如果脚本中定义解释器(bash等)，且脚本具有执行权限 +x，可以直接./shell.sh执行
# 如果脚本没有定义解释器，或者脚本不具备执行权限，或者不希望使用默认的shell执行脚本
# 可以使用 sh shell.sh 或者  zsh shell.sh
# 表示使用默认的shell执行啊本，使用zsh执行脚本

# Shell 注释
# 以 # 开头的行就是注释，会被解释器忽略。
# 如果在开发过程中，遇到大段的代码需要临时注释起来，过一会儿又取消注释，怎么办呢？
# 每一行加个#符号太费力了，可以把这一段要注释的代码用一对花括号括起来，定义成一个函数，没有地方调用这个函数，这块代码就不会执行，达到了和注释一样的效果。

# 多行注释
# 多行注释还可以使用以下格式：
:<<EOF
注释内容...
注释内容...
注释内容...
EOF
# EOF 也可以使用其他符号:
# 实例
:<<'
注释内容...
注释内容...
注释内容...
'
:<<!
注释内容...
注释内容...
注释内容...
!
```
# 定义变量，赋值，使用
## 定义
```shell
# 定义变量时，变量名不加美元符号（$，PHP语言中变量需要），如：

your_name="baidu.com"
# 注意，变量名和等号之间不能有空格，这可能和你熟悉的所有编程语言都不一样。同时，变量名的命名须遵循如下规则：

# 命名只能使用英文字母，数字和下划线，首个字符不能以数字开头。
# 中间不能有空格，可以使用下划线 _。
# 不能使用标点符号。
# 不能使用bash里的关键字（可用help命令查看保留关键字）。
# 有效的 Shell 变量名示例如下：
LD_LIBRARY_PATH
_var
var2
# 无效的变量命名：
?var=123
user*name=foo
```
## 赋值
```shell
# 除了显式地直接赋值，还可以用语句给变量赋值，如：

for file in `ls /etc`
# 或
for file in $(ls /etc)
# 以上语句将 /etc 下目录的文件名循环出来

```

## 使用
```shell
# 使用一个定义过的变量，只要在变量名前面加美元符号即可，如：

# 实例
your_name="qinjx"
echo $your_name
echo ${your_name}
# 变量名外面的花括号是可选的，加不加都行，加花括号是为了帮助解释器识别变量的边界，比如下面这种情况：

# 实例
for skill in Ada Coffe Action Java; do
    echo "I am good at ${skill}Script"
done
# 如果不给skill变量加花括号，写成echo "I am good at $skillScript"，解释器就会把$skillScript当成一个变量（其值为空），代码执行结果就不是我们期望的样子了。

# 推荐给所有变量加上花括号，这是个好的编程习惯。

# 已定义的变量，可以被重新定义，如：

# 实例
your_name="tom"
echo $your_name
your_name="alibaba"
echo $your_name
# 这样写是合法的，但注意，第二次赋值的时候不能写$your_name="alibaba"，使用变量的时候才加美元符（$）。

# 使用 readonly 命令可以将变量定义为只读变量，只读变量的值不能被改变。
# 下面的例子尝试更改只读变量，结果报错：
# 实例
myUrl="https://www.google.com"
readonly myUrl
myUrl="https://www.baidu.com"

# 使用 unset 命令可以删除变量。语法：
unset variable_name
```
变量类型
运行shell时，会同时存在三种变量：
1) **局部变量** 局部变量在脚本或命令中定义，仅在当前shell实例中有效，其他shell启动的程序不能访问局部变量。
2) **环境变量** 所有的程序，包括shell启动的程序，都能访问环境变量，有些程序需要环境变量来保证其正常运行。必要的时候shell脚本也可以定义环境变量。
3) **shell变量** shell变量是由shell程序设置的特殊变量。shell变量中有一部分是环境变量，有一部分是局部变量，这些变量保证了shell的正常运行

# 数据类型
## 字符串
```shell
# 字符串是shell编程中最常用最有用的数据类型（除了数字和字符串，也没啥其它类型好用了），字符串可以用单引号，也可以用双引号，也可以不用引号。

# 单引号
str='this is a string'
# 单引号字符串的限制：

# 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的；
# 单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用。
# 双引号
# 实例
your_name="foobar"
str="Hello, I know you are \"$your_name\"! \n"
echo -e $str

# 双引号的优点：
# 双引号里可以有变量
# 双引号里可以出现转义字符

# 拼接字符串
# 实例
your_name="foobar"
# 使用双引号拼接
greeting="hello, "$your_name" !"
greeting_1="hello, ${your_name} !"
echo $greeting  $greeting_1

# 使用单引号拼接
greeting_2='hello, '$your_name' !'
greeting_3='hello, ${your_name} !'
echo $greeting_2  $greeting_3
# 输出结果为：
# hello, foobar! hello, foobar!
# hello, foobar! hello, ${your_name} !

# 获取字符串长度
string="abcd"
echo ${#string}   # 输出 4

# 变量为数组时，${#string} 等价于 ${#string[0]}:
string="abcd"
echo ${#string[0]}   # 输出 4

# 提取子字符串
# 以下实例从字符串第 2 个字符开始截取 4 个字符：
string="foobar is a great site"
echo ${string:1:4} # 输出 ooba

# 查找字符 i 或 o 的位置(哪个字母先出现就计算哪个)：
string="foobar is a great site"
echo `expr index "$string" io`  # 输出 2
# 这里的起始位置从1开始，不是0

${string#*chars}	
# 从 string 字符串第一次出现 *chars 的位置开始，截取 *chars 右边的所有字符。
# ${string##*chars}	从 string 字符串最后一次出现 *chars 的位置开始，截取 *chars 右边的所有字符。
# ${string%*chars}	从 string 字符串第一次出现 *chars 的位置开始，截取 *chars 左边的所有字符。
# ${string%%*chars}	从 string 字符串最后一次出现 *chars 的位置开始，截取 *chars 左边的所有字符。
```
## 数组
bash支持一维数组（不支持多维数组），并且没有限定数组的大小。

类似于 C 语言，数组元素的下标由 0 开始编号。获取数组中的元素要利用下标，下标可以是整数或算术表达式，其值应大于或等于 0。
```shell
# 定义数组
# 在 Shell 中，用括号来表示数组，数组元素用"空格"符号分割开。定义数组的一般形式为：

# 数组名=(值1 值2 ... 值n)
array_name=(value0 value1 value2 value3)
# 或者
array_name=(
value0
value1
value2
value3
)
# 还可以单独定义数组的各个分量：
array_name[0]=value0
array_name[1]=value1
array_name[n]=valuen获取数组的长度

# 获取数组长度的方法与获取字符串长度的方法相同，例如：
# 取得数组元素的个数
length=${#array_name[@]}
# 或者
length=${#array_name[*]}
# 取得数组单个元素的长度
lengthn=${#array_name[n]}
# 可以不使用连续的下标，而且下标的范围没有限制。
# 读取数组
# 读取数组元素值的一般格式是：
# ${数组名[下标]}
# 例如：
valuen=${array_name[n]}
# 使用 @ 符号可以获取数组中的所有元素，例如：
echo ${array_name[@]}

```
# 加减乘除
Shell 和其他编程语言一样，支持多种运算符，包括：
- 算数运算符
- 关系运算符
- 布尔运算符
- 字符串运算符
- 文件测试运算符

## 算数运算符
原生bash不支持简单的数学运算，但是可以通过其他命令来实现，例如 awk 和 expr，expr 最常用。
expr 是一款表达式计算工具，使用它能完成表达式的求值操作。
```shell
#!/bin/bash
val=`expr 2 + 2`
echo "两数之和为 : $val"
# 两点注意：
# 表达式和运算符之间要有空格，例如 2+2 是不对的，必须写成 2 + 2，这与我们熟悉的大多数编程语言不一样。
# 完整的表达式要被 ` ` 包含，注意这个字符不是常用的单引号，在 Esc 键下边。
```
| 运算符 | 说明                                       | 举例                                  |
| :----- | :----------------------------------------- | :------------------------------------ |
| +      | 加法                                       | `expr $a + $b` 结果为 30。            |
| -      | 减法                                       | `expr $a - $b` 结果为 -10。           |
| *      | 乘法                                       | `expr $a \* $b` 结果为  200。         |
| /      | 除法                                       | `expr $b / $a` 结果为 2。             |
| %      | 取余                                       | `expr $b % $a` 结果为 0。             |
| =      | 赋值                                       | a=$b 把变量 b 的值赋给 a。            |
| ==     | 相等。 用于比较两个数字，相同则返回 true。 | [ $a == $b ] 返回 false。             |
| !=     | 不相等。                                   | 用于比较两个数字，不相同则返回 true。 | [ $a != $b ] 返回 true。 |

注意：条件表达式要放在方括号之间，并且要有空格，例如: [$a==$b] 是错误的，必须写成 [ $a == $b ]。

```bash
#!/bin/bash

a=10
b=20

val=`expr $a + $b`
echo "a + b : $val"

val=`expr $a - $b`
echo "a - b : $val"

val=`expr $a \* $b`
echo "a * b : $val"

val=`expr $b / $a`
echo "b / a : $val"

val=`expr $b % $a`
echo "b % a : $val"

if [ $a == $b ]
then
   echo "a 等于 b"
fi
if [ $a != $b ]
then
   echo "a 不等于 b"
fi

:<<EOF
Result:
a + b : 30
a - b : -10
a * b : 200
b / a : 2
b % a : 0
a 不等于 b
EOF
```
注意：
乘号(\*)前边必须加反斜杠(\)才能实现乘法运算；
if...then...fi 是条件语句，后续将会讲解。
在 MAC 中 shell 的 expr 语法是：$((表达式))，此处表达式中的 "*" 不需要转义符号 "\" 


## 关系运算符
关系运算符只支持数字，不支持字符串，除非字符串的值是数字。
下表列出了常用的关系运算符，假定变量 a 为 10，变量 b 为 20：
| 运算符 | 说明                                                  | 举例                       |
| :----- | :---------------------------------------------------- | :------------------------- |
| -eq    | 检测两个数是否相等，相等返回 true。                   | [ $a -eq $b ] 返回 false。 |
| -ne    | 检测两个数是否不相等，不相等返回 true。               | [ $a -ne $b ] 返回 true。  |
| -gt    | 检测左边的数是否大于右边的，如果是，则返回 true。     | [ $a -gt $b ] 返回 false。 |
| -lt    | 检测左边的数是否小于右边的，如果是，则返回 true。     | [ $a -lt $b ] 返回 true。  |
| -ge    | 检测左边的数是否大于等于右边的，如果是，则返回 true。 | [ $a -ge $b ] 返回 false。 |
| -le    | 检测左边的数是否小于等于右边的，如果是，则返回 true。 | [ $a -le $b ] 返回 true。  |

```bash
#!/bin/bash

a=10
b=20

if [ $a -eq $b ]
then
   echo "$a -eq $b : a 等于 b"
else
   echo "$a -eq $b: a 不等于 b"
fi
if [ $a -ne $b ]
then
   echo "$a -ne $b: a 不等于 b"
else
   echo "$a -ne $b : a 等于 b"
fi
if [ $a -gt $b ]
then
   echo "$a -gt $b: a 大于 b"
else
   echo "$a -gt $b: a 不大于 b"
fi
if [ $a -lt $b ]
then
   echo "$a -lt $b: a 小于 b"
else
   echo "$a -lt $b: a 不小于 b"
fi
if [ $a -ge $b ]
then
   echo "$a -ge $b: a 大于或等于 b"
else
   echo "$a -ge $b: a 小于 b"
fi
if [ $a -le $b ]
then
   echo "$a -le $b: a 小于或等于 b"
else
   echo "$a -le $b: a 大于 b"
fi
:<<EOF
10 -eq 20: a 不等于 b
10 -ne 20: a 不等于 b
10 -gt 20: a 不大于 b
10 -lt 20: a 小于 b
10 -ge 20: a 小于 b
10 -le 20: a 小于或等于 b
EOF
```

## 布尔运算符
下表列出了常用的布尔运算符，假定变量 a 为 10，变量 b 为 20：

| 运算符 | 说明                                                | 举例                                     |
| :----- | :-------------------------------------------------- | :--------------------------------------- |
| !      | 非运算，表达式为 true 则返回 false，否则返回 true。 | [ ! false ] 返回 true。                  |
| -o     | 或运算，有一个表达式为 true 则返回 true。           | [ $a -lt 20 -o $b -gt 100 ] 返回 true。  |
| -a     | 与运算，两个表达式都为 true 才返回 true。           | [ $a -lt 20 -a $b -gt 100 ] 返回 false。 |

```bash
#!/bin/bash

a=10
b=20

if [ $a != $b ]
then
   echo "$a != $b : a 不等于 b"
else
   echo "$a == $b: a 等于 b"
fi
if [ $a -lt 100 -a $b -gt 15 ]
then
   echo "$a 小于 100 且 $b 大于 15 : 返回 true"
else
   echo "$a 小于 100 且 $b 大于 15 : 返回 false"
fi
if [ $a -lt 100 -o $b -gt 100 ]
then
   echo "$a 小于 100 或 $b 大于 100 : 返回 true"
else
   echo "$a 小于 100 或 $b 大于 100 : 返回 false"
fi
if [ $a -lt 5 -o $b -gt 100 ]
then
   echo "$a 小于 5 或 $b 大于 100 : 返回 true"
else
   echo "$a 小于 5 或 $b 大于 100 : 返回 false"
fi

:<<EOF
10 != 20 : a 不等于 b
10 小于 100 且 20 大于 15 : 返回 true
10 小于 100 或 20 大于 100 : 返回 true
10 小于 5 或 20 大于 100 : 返回 false
EOF
```
## 逻辑运算符
以下介绍 Shell 的逻辑运算符，假定变量 a 为 10，变量 b 为 20:

| 运算符 | 说明       | 举例                                      |
| :----- | :--------- | :---------------------------------------- |
| &&     | 逻辑的 AND | [[ $a -lt 100 && $b -gt 100 ]] 返回 false |
| \|\|   | 逻辑的 OR  | [[ $a -lt 100                             |  | $b -gt 100 ]] 返回 true |

```bash
#!/bin/bash

a=10
b=20

if [[ $a -lt 100 && $b -gt 100 ]]
then
   echo "返回 true"
else
   echo "返回 false"
fi

if [[ $a -lt 100 || $b -gt 100 ]]
then
   echo "返回 true"
else
   echo "返回 false"
fi
:<<EOF
返回 false
返回 true
EOF
```
## 字符串运算符
下表列出了常用的字符串运算符，假定变量 a 为 "abc"，变量 b 为 "efg"：

| 运算符 | 说明                                         | 举例                     |
| :----- | :------------------------------------------- | :----------------------- |
| =      | 检测两个字符串是否相等，相等返回 true。      | [ $a = $b ] 返回 false。 |
| !=     | 检测两个字符串是否不相等，不相等返回 true。  | [ $a != $b ] 返回 true。 |
| -z     | 检测字符串长度是否为0，为0返回 true。        | [ -z $a ] 返回 false。   |
| -n     | 检测字符串长度是否不为 0，不为 0 返回 true。 | [ -n "$a" ] 返回 true。  |
| $      | 检测字符串是否为空，不为空返回 true。        | [ $a ] 返回 true。       |

```bash
#!/bin/bash

a="abc"
b="efg"

if [ $a = $b ]
then
   echo "$a = $b : a 等于 b"
else
   echo "$a = $b: a 不等于 b"
fi
if [ $a != $b ]
then
   echo "$a != $b : a 不等于 b"
else
   echo "$a != $b: a 等于 b"
fi
if [ -z $a ]
then
   echo "-z $a : 字符串长度为 0"
else
   echo "-z $a : 字符串长度不为 0"
fi
if [ -n "$a" ]
then
   echo "-n $a : 字符串长度不为 0"
else
   echo "-n $a : 字符串长度为 0"
fi
if [ $a ]
then
   echo "$a : 字符串不为空"
else
   echo "$a : 字符串为空"
fi
:<<EOF
abc = efg: a 不等于 b
abc != efg : a 不等于 b
-z abc : 字符串长度不为 0
-n abc : 字符串长度不为 0
abc : 字符串不为空
EOF
```

## 文件测试运算符
文件测试运算符用于检测 Unix 文件的各种属性。

属性检测描述如下：

| 操作符  | 说明                                                                        | 举例                      |
| :------ | :-------------------------------------------------------------------------- | :------------------------ |
| -b file | 检测文件是否是块设备文件，如果是，则返回 true。                             | [ -b $file ] 返回 false。 |
| -c file | 检测文件是否是字符设备文件，如果是，则返回 true。                           | [ -c $file ] 返回 false。 |
| -d file | 检测文件是否是目录，如果是，则返回 true。                                   | [ -d $file ] 返回 false。 |
| -f file | 检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。 | [ -f $file ] 返回 true。  |
| -g file | 检测文件是否设置了 SGID 位，如果是，则返回 true。                           | [ -g $file ] 返回 false。 |
| -k file | 检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。                 | [ -k $file ] 返回 false。 |
| -p file | 检测文件是否是有名管道，如果是，则返回 true。                               | [ -p $file ] 返回 false。 |
| -u file | 检测文件是否设置了 SUID 位，如果是，则返回 true。                           | [ -u $file ] 返回 false。 |
| -r file | 检测文件是否可读，如果是，则返回 true。                                     | [ -r $file ] 返回 true。  |
| -w file | 检测文件是否可写，如果是，则返回 true。                                     | [ -w $file ] 返回 true。  |
| -x file | 检测文件是否可执行，如果是，则返回 true。                                   | [ -x $file ] 返回 true。  |
| -s file | 检测文件是否为空（文件大小是否大于0），不为空返回 true。                    | [ -s $file ] 返回 true。  |
| -e file | 检测文件（包括目录）是否存在，如果是，则返回 true。                         | [ -e $file ] 返回 true。  |

其他检查符：
-S: 判断某文件是否 socket。
-L: 检测文件是否存在并且是一个符号链接。

```bash
#!/bin/bash

file="/var/www/foobar/test.sh"
if [ -r $file ]
then
   echo "文件可读"
else
   echo "文件不可读"
fi
if [ -w $file ]
then
   echo "文件可写"
else
   echo "文件不可写"
fi
if [ -x $file ]
then
   echo "文件可执行"
else
   echo "文件不可执行"
fi
if [ -f $file ]
then
   echo "文件为普通文件"
else
   echo "文件为特殊文件"
fi
if [ -d $file ]
then
   echo "文件是个目录"
else
   echo "文件不是个目录"
fi
if [ -s $file ]
then
   echo "文件不为空"
else
   echo "文件为空"
fi
if [ -e $file ]
then
   echo "文件存在"
else
   echo "文件不存在"
fi
:<<EOF
文件可读
文件可写
文件可执行
文件为普通文件
文件不是个目录
文件不为空
文件存在
EOF
```
# 判断
在 sh/bash ，如果 else 分支没有语句执行，就不要写这个 else。因为代码块里面不允许为空。
```bash
# 单if结构
if condition
then
    command1 
    command2
    ...
    commandN 
fi
if [ $(ps -ef | grep -c "ssh") -gt 1 ]; then echo "true"; fi

# if else 结构
if condition
then
    command1 
    command2
    ...
    commandN
else
    commandM
fi

# if elif else 结构
if condition1
then
    command1
elif condition2 
then 
    command2
else
    commandN
fi

# 举例
a=10
b=20
if [ $a == $b ]
then
   echo "a 等于 b"
elif [ $a -gt $b ]
then
   echo "a 大于 b"
elif [ $a -lt $b ]
then
   echo "a 小于 b"
else
   echo "没有符合的条件"
fi
```
# 循环
## for
```bash
# 脚本格式
for var in item1 item2 ... itemN
do
    command1
    command2
    ...
    commandN
done
# 命令行格式
for var in item1 item2 ... itemN; do command1; command2… done;
```
当变量值在列表里，for 循环即执行一次所有命令，使用变量名获取列表中的当前取值。命令可为任何有效的 shell 命令和语句。in 列表可以包含替换、字符串和文件名。

in列表是可选的，如果不用它，for循环使用命令行的位置参数。

例如，顺序输出当前列表中的数字：
```bash
for loop in 1 2 3 4 5
do
    echo "The value is: $loop"
done

:<<EOF
结果：
The value is: 1
The value is: 2
The value is: 3
The value is: 4
The value is: 5
EOF
```

```bash
# 顺序输出字符串中的字符：
#!/bin/bash

for str in This is a string
do
    echo $str
done

# 输出结果：
# This
# is
# a
# string
```

还有其他格式，如下：
```bash
for i in {1..10}
do
    echo $i
done

# 输出结果
# 1
# 2
# ...
# 10
```

```bash
for ((i=1; i<=5; i++))
do
    # 使用printf函数来格式化数字为三位数，不足三位的数字会在前面补零
    number=$(printf "%03d" $i)
    echo "Tester$number"
done

# 输出结果
# Tester001
# Tester002
# Tester003
# Tester004
# Tester005
```

## while
```bash
while condition
do
    command
done
```
```bash
以下是一个基本的 while 循环，测试条件是：如果 int 小于等于 5，那么条件返回真。int 从 1 开始，每次循环处理时，int 加 1。运行上述脚本，返回数字 1 到 5，然后终止。

实例
#!/bin/bash
int=1
while(( $int<=5 ))
do
    echo $int
    let "int++"
done

:<<EOF
运行脚本，输出：
1
2
3
4
5
EOF

:<<EOF
let 命令是 BASH 中用于计算的工具，用于执行一个或多个表达式，变量计算中不需要加上 $ 来表示变量。如果表达式中包含了空格或其他特殊字符，则必须引起来。

语法格式
let arg [arg ...]
参数说明：
arg：要执行的表达式

实例：
自加操作：let no++
自减操作：let no--
简写形式 let no+=10，let no-=20，分别等同于 let no=no+10，let no=no-20。

以下实例计算 a 和 b 两个表达式，并输出结果：
#!/bin/bash

let a=5+4
let b=9-3 
echo $a $b
以上实例执行结果为：

9 6
EOF
```
## 死循环
```bash
while :
do
    command
don

while true
do
    command
done

for (( ; ; ))
```

## until
until 循环执行一系列命令直至条件为 true 时停止。
until 循环与 while 循环在处理方式上刚好相反。
一般 while 循环优于 until 循环，但在某些时候—也只是极少数情况下，until 循环更加有用。
until 语法格式:
```bash
until condition
do
    command
done
```
condition 一般为条件表达式，如果返回值为 false，则继续执行循环体内的语句，否则跳出循环。
```bash
#!/bin/bash

a=0

until [ ! $a -lt 10 ]
do
   echo $a
   a=`expr $a + 1`
done
```

## case ... esac
case ... esac 为多选择语句，与其他语言中的 switch ... case 语句类似，是一种多分支选择结构，每个 case 分支用右圆括号开始，用两个分号 ;; 表示 break，即执行结束，跳出整个 case ... esac 语句，esac（就是 case 反过来）作为结束标记。

可以用 case 语句匹配一个值与一个模式，如果匹配成功，执行相匹配的命令。

case ... esac 语法格式如下：
```bash
case 值 in
模式1)
    command1
    command2
    ...
    commandN
    ;;
模式2)
    command1
    command2
    ...
    commandN
    ;;
esac
```
case 工作方式如上所示，取值后面必须为单词 in，每一模式必须以右括号结束。取值可以为变量或常数，匹配发现取值符合某一模式后，其间所有命令开始执行直至 ;;。

取值将检测匹配的每一个模式。一旦模式匹配，则执行完匹配模式相应命令后不再继续其他模式。如果无一匹配模式，使用星号 * 捕获该值，再执行后面的命令。

## 跳出循环
在循环过程中，有时候需要在未达到循环结束条件时强制跳出循环，Shell使用两个命令来实现该功能：break和continue。

break命令
break命令允许跳出所有循环（终止执行后面的所有循环）。

下面的例子中，脚本进入死循环直至用户输入数字大于5。要跳出这个循环，返回到shell提示符下，需要使用break命令。
```bash
#!/bin/bash
while :
do
    echo -n "输入 1 到 5 之间的数字:"
    read aNum
    case $aNum in
        1|2|3|4|5) echo "你输入的数字为 $aNum!"
        ;;
        *) echo "你输入的数字不是 1 到 5 之间的! 游戏结束"
            break
        ;;
    esac
done
```

continue
continue命令与break命令类似，只有一点差别，它不会跳出所有循环，仅仅跳出当前循环。

对上面的例子进行修改：
```bash
#!/bin/bash
while :
do
    echo -n "输入 1 到 5 之间的数字: "
    read aNum
    case $aNum in
        1|2|3|4|5) echo "你输入的数字为 $aNum!"
        ;;
        *) echo "你输入的数字不是 1 到 5 之间的!"
            continue
            echo "游戏结束"
        ;;
    esac
done
```
# 时间处理
时间从date命令中取值，但是回显的格式往往并不符合我们的预期。而涉及到时间的处理，常用的是时间字符串与时间戳的转换，时间戳的大小比较。
```bash
# 获取指定格式的时间(字符串)
date +"%Y-%m-%d %H:%M:%S"
# 年-月-日 时:分:秒

# 获取时间戳
date +"%s"
# 1652000993
```
- 字符串转时间戳
	```bash
	date -d "2010-10-18 00:00:00" +%s 
	# 1287360000
	```
- 时间戳转字符串
	```bash
	date -d @1287331200  "+%Y-%m-%d %H:%M:%S"
	# 2010-10-17 16:00:00
	```
- 时间戳比较大小
	```bash
	t1=$(date +%s)
	sleep 2
	t2=$(date +%s)
	echo $((t2-t1))
	```
# 正则表达式
后续单独整理一篇文章