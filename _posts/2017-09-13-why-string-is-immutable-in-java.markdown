---
layout:     post
title:      "为什么Java中的String类是不可变的？"
subtitle:   "Why String is immutable in Java ?"
date:       2017-09-13
author:     "Jon Lee"
header-img: "img/in-post/2017-09-13-why-string-is-immutable-in-java/bg.jpg"
catalog:    false
categories : Java基础
tags:
    - Java
---
---
### 什么是不可变类

String类是Java中的一个 **不可变类**（immutable class）。  
简单来说，不可变类就是实例在被创建之后不可修改。  
在《Effective Java》 Item 15 中提到了为了使类成为不可变，需要遵循的五条规则：

1. 不要提供任何会修改对象状态的方法。
2. 保证类不会被扩展。
3. 使所有的域都是final的。
4. 使所有域都成为私有的。
5. 确保对于任何可变组件的互斥访问。  

不可变类有许多优点，不可变类比可变类更加易于设计、实现和使用，不容易出错，且更加安全。  
下面来看一下Java把String类设计为不可变类的几个原因：

### 由于字符串池的存在

由于字符串常量池的存在，两个字符串可能指向同一个对象。

    String string1 = "abcd";
    String string2 = "abcd";

![](/img/in-post/2017-09-13-why-string-is-immutable-in-java/1.jpg)

如果字符串是可变的话，通过一个引用改变它的值，将会导致其他引用的值也同样改变，从而可能发生错误。

### 缓存hashcode

Java中经常用到一个字符串的 **hashcode**，例如在HashMap和HashSet中。  
不可变性保证同一个字符串对象的hashcode总是相同的，而在使用时不用考虑其是否发生改变。  
这意味着不需要每次都计算一遍hashcode，使程序更加高效。

    private int hash; // Default to 0

### 保证其他对象的使用

为了具体一点，举个🌰。

    HashSet<String> set = new HashSet<String>();
    set.add(new String("a"));
    set.add(new String("b"));
    set.add(new String("c"));

    for(String s : set)
            s.value = "a";

 在这个例子中，如果String是可变的话将破坏HashSet（集合中可能会包含重复元素）。

### 安全性

在Java中String被用作许多方法的参数，例如网络连接，对文件的操作等等。  
假如String不是不可变的，一个连接或文件将可能被改变，这会产生严重的安全隐患。  
因为Java中反射的参数也是字符串，所以可变字符串同样会产生安全问题。

下面是一个例子：

    boolean connect(string s){
            if (!isSecure(s)) {
                throw new SecurityException();
            }
            // here will cause problem, if s is changed before this by using other references.    
            causeProblem(s);
    }

### 不可变对象是线程安全的

不可变对象是线程安全的，它们可以被自由地共享，不要求同步。

---
不可变类的优势有很多，当然也有缺点，就是对于每个不同的值都需要一个单独的对象。

### 参考资料

>《Effective Java Ⅱ》 Item 15  
https://www.programcreek.com/2013/04/why-string-is-immutable-in-java/
