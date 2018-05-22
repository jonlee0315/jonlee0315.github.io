---
layout:     post
title:      "Markdown基本语法"
subtitle:   "How to M↓"
date:       2017-05-21
author:     "Jon Lee"
header-img: "img/in-post/2018-05-21-learn-markdown/bg.jpg"
catalog: true
categories : 随笔
tags:
    - Markdown
---

# 概述
Markdown是一种轻量级标记语言，创始人为John Gruber，大部分语法是和Aaron Swartz共同合作完成。它允许人们“使用易读易写的纯文本格式编写文档，然后转换成有效的HTML文档”。
# 区块元素
#### 段落和换行
段落的前后要有空行，所谓的空行是指没有文字内容。  
若想在段内强制换行的方式是使用两个以上空格加上回车。
#### 标题
Markdown支持两种标题语法，类Setext和类Atx形式。
类 Setext 形式是用底线的形式，利用`=`（一级标题）和`-`（二级标题），例如：

    一级标题  
    ======  
    二级标题  
    ------

任意数量的`=`和`-`都可以。  
类Atx形式是在行首插入1到6个`#`，对应一到六级标题，例如：

    # 一级标题
    ## 二级标题    
    ### 三级标题    
    #### 四级标题    
    ##### 五级标题

#### 区块引用
在段落首行开头使用`>`符号，也可以嵌套使用。

    >区块引用
    >>嵌套引用

效果：
>区块引用
>>嵌套引用

引用的区块内也可以使用其它的Markdown语法，包括标题、列表、代码区块等。

#### 列表
无序列表使用`*`、`+`和`-`作为列表标记：

    * Java
    * C++
    * Python

效果：
* Java
* C++
* Python

有序列表使用数字加上`.`。

    1. Java
    2. C++
    3. Python

效果：
1. Java
2. C++
3. Python

#### 代码区块
使用4个空格或者1个制表符实现代码区块。

    这是一个代码区块。

#### 分割线
在一行中使用三个以上的`*`、`-`或`_`来建立一个分割线。

    ******
    ------
    ______

效果：

******
------
______

# 区段元素

#### 链接
Markdown支持两种形式的链接语法：行内式和参考式。  
两种方式都需要用`[ ]`将链接文字包裹起来。  
- 行内式：
    语法为：`[link text](url "title")`。

        这是一个[行内式](lijiaheng.me "点我")链接

    效果：  
    这是一个[行内式](lijiaheng.me "点我")链接

- 参考式：
    语法为：`[link text][id]`,然后在任意处进行定义`[id]: url "title"`（不能在同一行中）。

        这是一个[参考式][1]链接
        [1]: lijiaheng.me "摸我"

    效果：  
    这是一个[参考式][id]链接

    [id]: lijiaheng.me "摸我"

#### 强调
使用`*`和`_`来包裹被强调的内容

    *斜体*
    _斜体_
    **强调**
    __强调__

效果：  
*斜体*  
_斜体_  
**强调**  
__强调__

#### 代码
使用'\`'包裹代码，来达到标记效果。

    \`这是一段代码\`

效果：`这是一段代码`

#### 图片
图片的插入和链接的语法几乎相同，区别是在开头加一个`!`。
* 行内式

        ![This is an image](/img/in-post/2018-05-21-learn-markdown/1.png "title")

    效果：
    ![This is an image](/img/in-post/2018-05-21-learn-markdown/1.png "title")
* 参考式

        ![This is an image][1]
        [1]: /img/in-post/2018-05-21-learn-markdown/1.png "title"

    效果：
    ![This is an image][1]

    [1]: /img/in-post/2018-05-21-learn-markdown/1.png "title"


# 其他

#### 表格
使用`|`分隔单元格，`-`分隔表头和其他行：

    name | age | gender
    ---- | --- | ------
    Jon Lee | 24 | male
    Yoetsu  |24  | female

效果：

name | age | gender
---- | --- | ------
Jon Lee | 24 | male
Yoetsu  |24  | female

#### 反斜杠
转义字符`\`，下列符号需要转义：

        \   反斜线
        `   反引号
        *   星号
        \_   底线
        {}  花括号
        []  方括号
        ()  括弧
        #   井字号
        +   加号
        -   减号
        .   英文句点
        !   惊叹号

#### 数学公式
支持Latex语法

        $$E=mc^2$$
        $$x = {-b \pm \sqrt{b^2-4ac} \over 2a}$$

效果：

$$E=mc^2$$

$$x = {-b \pm \sqrt{b^2-4ac} \over 2a}$$

# References

> <http://www.markdown.cn/>  
> <https://github.com/younghz/Markdown>

---
