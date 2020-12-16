---
title: Markdown 常用语法备忘
date: 2019-05-09T18:25:56+08:00
draft: false
description: Markdown 常用语法备忘
tags: [ "Markdown" ]
keywords: [ "Markdown" ]
categories: [ "分享" ]
isCJKLanguage: true
enableMathJax: true
enableDisqus: true
---

## 标题 ##

``` markdown
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```
> 也可以使用闭合方式的标题，结尾的`#`可以不必和开头一致
``` markdown
# 一级标题 #
## 二级标题 ##
...
```
另一种方式
``` markdown
一级标题
=======

二级标题
-------
```
当然也可以用**HTML**的方式

``` html
<h1>一级标题</h1>
<h2>二级标题</h2>
<h3>三级标题</h3>
<h4>四级标题</h4>
<h5>五级标题</h5>
<h6>六级标题</h6>
```
**HTML**的好处在于可以方便的使标题居中
``` html
<h1 align="center">居中标题</h1>
```

## 目录 ##

可以使用`[TOC]`标记来自动生成目录，但兼容性貌似不怎么好

``` markdown
[TOC]
```

## 分隔线 ##

> 可以使用3个以上的`*`、`-`作为分隔线，中间也可以插入`空格`

``` markdown
***
* * *
---
- - -
```

## 字体 ##

**粗体**

在需要以粗体显示的文字前后各加两个`*`或`_`可以使文字加粗显示

``` markdown
**粗体**
__粗体__
```

*斜体*

在需要以斜体显示的文字前后各加一个`*`或`_`可以使文字已斜体显示

``` markdown
*斜体*
_斜体_
```

~~删除线~~

在文字前后各加两个`~`可以在文字上添加删除线

``` markdown
~~删除线~~
```

当然也可以进行组合使用

``` markdown
***斜体加粗***
__~~粗体删除线~~__
```

颜色
在写作过程中可能会遇到不少情况需要将文字用不同颜色标注，可以使用**HTML**的方式来实现，同时也可以设置字体和大小

``` html
<font face="微软雅黑" color=red size=12>落霞与孤鹜齐飞，秋水共长天一色。</font>
```

## 段落 ##

Markdown的换行有些奇特，直接`Enter`换行它好像不认，需要在段落结尾加**两个空格+换行**才可以，或者在上一段落和下一段落之间再加一行空行，即**两次换行**也可以。

<font color='#FF69B4'>落霞与孤鹜齐飞，秋水共长天一色。</font>
渔舟唱晚，响穷彭蠡之滨；

雁阵惊寒，声断衡阳之浦。

``` markdown
落霞与孤鹜齐飞，秋水共长天一色。
渔舟唱晚，响穷彭蠡之滨；

雁阵惊寒，声断衡阳之浦。
```

## 引用 ##

写在`>`后的文字即可显示为引用，引用可以嵌套使用

``` markdown
> 引用的文字
>> 嵌套引用的文字
>>> 更多嵌套
```

## 表格 ##

|表头|表头|表头|表头|
|---|:--|:--:|---:|
|内容|居左|居中|居右|

``` markdown
|表头|表头|表头|表头|
|---|:--|:--:|---:|
|内容|居左|居中|居右|
```

第一行是表头，第二行代表对齐方式，默认是**居左**，在`-`左边加`:`即可**居左**对齐，在`-`右边加`:`可**居右对齐**，两边都加`:`表示**居中**对齐

## 列表 ##

**有序列表**

``` markdown
1. 列表1
2. 列表2
3. 列表3
```

**无序列表**

可以使用`*`、`+`或者`-`作为标记
``` markdown
* 列表1
+ 列表2
- 列表3
```

**任务列表**

- [x] @mentions, #refs, [links](), **formatting**, and <del>tags</del> supported
- [x] list syntax required (any unordered or ordered list supported)
- [x] this is a complete item
- [ ] this is an incomplete item

``` markdown
- [x] 已完成的任务
- [ ] 未完成的任务
```

## 链接 ##

可以直接输入网址，如：https://github.com/

或者使用格式：`[Text](url)`

点击<a href="https://kira-96.github.io/" target="_blank">这里</a>返回主页

``` markdown
点击[这里](https://kira-96.github.io/)返回主页
```

也可以使用**HTML**的方式

``` html
点击<a href="https://kira-96.github.io/" target="_blank">这里</a>返回主页
```

还有一种就是使用索引的方式
例：[谷歌][1]、[百度][2]

[1]: https://www.google.com.hk/ "google"
[2]: https://www.baidu.com/ "百度"

``` markdown
例：[谷歌][1]、[百度][2]

[1]: https://www.google.com.hk/ "google"
[2]: https://www.baidu.com/ "百度"
```

## 锚 ##

主要用于在页面内跳转

点击[这里](#链接)查看链接的用法

``` markdown
点击[这里](#链接)查看链接的用法
```


## 图片 ##

图片和链接的格式很像，url可以使用相对位置和绝对位置，当然网络位置也可以

`![Alt Text](url)`

``` markdown
![图片](https://image-url.jpg)
```

也可以使用**HTML**的方式

``` html
<img src="https://image-url.jpg" width="50%" height="50%">
```

设置对齐方式

``` html
<div align=center>
    <img src="https://image-url.jpg" width="50%" height="50%">
</div>
```

## 标注 ##

这个用的并不多，看起来像是课本上文言文里面那种注释的感觉

例：

滕王阁序的作者是王勃[^1]。

[^1]: 王勃（约650——676年），唐代诗人。汉族，字子安。绛州龙门(今山西河津)人。王勃与杨炯、卢照邻、骆宾王齐名，世称“初唐四杰”，其中王勃是“初唐四杰”之首。

``` markdown
滕王阁序的作者是王勃[^1]。

[^1]: 王勃（约650——676年），唐代诗人。汉族，字子安。绛州龙门(今山西河津)人。王勃与杨炯、卢照邻、骆宾王齐名，世称“初唐四杰”，其中王勃是“初唐四杰”之首。
```

## 行内代码 ##

可以直接使用两个`（反引号）包裹行内代码

例：我们学习的第一行代码通常都是`printf("Hello World!")`。

``` markdown
我们学习的第一行代码通常都是`printf("Hello World!")`。
```

## 语法高亮 ##

``` cpp
int main(void)
{
    printf("Hello World!\n");
    return 0;
}
```

``` markdown
​``` cpp
int main(void)
{
    printf("Hello World!\n");
    return 0;
}
​```
```

## 公式

公式对于写论文的同学来说是非常有用的，Markdown的公式也比word的公式编辑方便多了。

行内公式，使用`$ $`包括在内。

如：$e=mc^2$

``` markdown
$e=mc^2$
```

单行公式，公式会单独占用一行，使用`$$ $$`包括在内。

$$Fe+CuSO_4=FeSO_4+Cu$$

``` markdown
$$Fe+CuSO_4=FeSO_4+Cu$$
```

其中具体的符号和字母之类的需要的时候可以到网上去找，如[Markdown 数学公式](https://geek-docs.com/markdown/markdown-tutorial/markdown-mathematical-formula.html)。

## 转义字符 ##

``` markdown
\\ 反斜杠

\` 反引号

\* 星号

\_ 下划线

\{\} 大括号

\[\] 中括号

\(\) 小括号

\# 井号

\+ 加号

\- 减号

\. 英文句号

\! 感叹号
```

## 注释 ##

可以使用**HTML**的注释方式，会在生成的**HTML**中以注释的形式存在，不显示出来。

<!-- 我是注释内容 -->

``` html
<!-- 我是注释内容 -->
```

或者使用

<div style='display: none'>
    我是注释内容
</div>

``` html
<div style='display: none'>
    我是注释内容
</div>
```

## :smiley: Emoji​ :tada: ##

Markdown甚至支持**Emoji**

:heart_eyes::stuck_out_tongue_winking_eye::angry::anger::mask::imp::smiling_imp::two_hearts:

[Emoji Cheat Sheet](https://github.com/ikatyang/emoji-cheat-sheet/blob/master/README.md)

## 写在最后 ##

自从接触了Markdown之后，我就很少使用Word这类工具了。日常工作和生活中用它来写文档和笔记真的是相当舒服，语法简单好记，完全可以满足需求，使用起来方便快捷，还可以借助**HTML**来实现一些比较复杂的功能。

不过我们公司内部似乎没什么人使用，可能是由于我们公司并不是互联网企业，所以没有那么潮流，感觉可以借机会安利一波，对于提高整体的工作效率也有不小的帮助。
