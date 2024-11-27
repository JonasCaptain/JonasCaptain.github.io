---
layout: "@layouts/ArticleLayout.astro"
title: 使用纯CSS获取视口分辨率
description: 仅仅使用CSS来获取视口的分辨率
date: 2024-11-27 14:26:36
tags: 
    - 前端
    - CSS
---

使用如下的html即可完成，非常神奇

```html
<html>
<head>
    <title>获取视口像素</title>
    <style>
        <!-->
        @property用于自定义一个CSS属性。--属性名，是标准格式，但在使用的时候，需要用var(--属性名)；
        @property定义时，用到了3个字段：
            syntax：该属性用什么方式解析，枚举值有很多，见：https://drafts.css-houdini.org/css-properties-values-api/#supported-names
            inherits：true/false，表示该属性是否允许被继承
            initial-value：该属性的初始值
        <-->
    
        @property --vh {
            syntax: '<length>';
            inherits: true;
            initial-value: 100vh;
        }
        @property --vw {
            syntax: '<length>';
            inherits: true;
            initial-value: 100vw;
        }
        <!-->
        :root 是一个伪类，对于HTML来说，:root就表示html元素
        此处表示在html元素上，定义两个变量--w、--h，这两个变量是使用--vw、--vh经过三角函数的计算得到的；
        atan2是反正切函数，tan是正切函数；
        <-->
        :root {
            --w: tan(atan2(var(--vw), 1px));
            --h: tan(atan2(var(--vh), 1px));
        }
        <!-->
        body::before 表示在body标签之前添加一个伪元素
        content 表示该伪元素显示的文本内容
        counter-reset 重置2个CSS计数器，w 和 h 初始值分别为 --w 和 --h
        font-size 定义文本字体大小
        font-weight 定义文本字体粗细
        position：fixed 固定定位
        inset: 0  inset用于定位元素指定距离，且它是一个简写形式对应(top right bottom left)
                传1个值时，就对应4个方向；传2个值，就对应top bottom；传4个值，就分别对应4个方向
        width: fit-content  使元素的宽度适应其内容的大小
        height: fit-content  使元素的高度适应其内容的大小
        margin: auto 外边距自适应
        <-->
        body::before {
            content: counter(w) " x " counter(h);
            counter-reset: w var(--w) h var(--h);
            font-size: 150px;
            font-weight: 900;
            position: fixed;
            inset: 0;
            width: fit-content;
            height: fit-content;
            margin: auto;
        }
    </style>
</head>
<body>
</body>
</html>
```