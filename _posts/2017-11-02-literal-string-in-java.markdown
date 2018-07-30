---
layout:     post
title:      "[译文] Java中的字符串字面量"
subtitle:   "Strings, Literally"
date:       2017-11-02
author:     "Jon Lee"
header-img: "img/in-post/2017-11-02-literal-string-in-java/bg.jpg"
catalog:    false
categories : Java基础
tags:
    - Java
---

>让我们由一个简单的问题开始，什么是字符串字面量？一个字符串字面量就是两个双引号之间的字符序列，形如 **"string"** 、**"literal"**。  
你可能已经在你的程序中使用字符串字面量几百次了，但是你可能还没意识到它在Java中是多么特殊。

### 字符串是不可变的

究竟什么使字符串字面量这么特殊？首先，记住重要的一点是字符串对象是不可变的。  
这就意味着一旦创建，一个字符串对象就不能被改变（还是可以通过反射来改变）。

不可变？不能被更改？那怎么解释这段代码。

    public class ImmutableStrings {
            public static void main(String[] args) {
                    String start = "Hello";
                    String end = start.concat(" World!");
                    System.out.println(end);
            }
    }
    // Output
    Hello World!

看这段代码，字符串被改变了吗，还是没有？事实上，这段代码中并没有字符串对象被改变。

我们首先将"Hello"赋值给`start`变量，为了实现这步，需要在堆中创建一个对象，并把它的引用存储在`start`中。接下来，我们在这个对象上调用`concat(String)`方法。进行到这里Java耍了一个小把戏，如果我们查看String的API说明，会发现其中对于`concat(String)`方法有如下的描述：

![](/img/in-post/2017-11-02-literal-string-in-java/1.jpg)

>方法描述：将指定字符串连接在这个字符串的结尾。  
如果长度为0，则返回这个字符串对象。否则就创建一个新的字符串对象，表示这个字符串序列由原字符串对象和参数字符串二者所表示的字符串序列拼接而成。

你肯定看到了，当你将两个字符串做拼接操作时，实际上并没有改变原对象，而是直接创建了一个包含原始对象的新的对象，并且将另一个字符串拼在了后面。

我们上面那段代码就是这么执行的，`start`变量所引用的字符串对象并没有改变，如果在调用`concat`方法之后`System.out.println(start);`，会发现`start`仍然指向的是"Hello"。

这时候你可能想到了字符串中的“+”操作符，事实上字符串的`+`操作也是和concat做了同样的事情（`+`操作实际上是new了一个`StringBuilder`对象，然后调用`append`方法）。

### 字符串的存储——字符串常量池

你或许听说过“字符串常量池”这个概念，究竟什么是字符串常量池？有人说是一个字符串对象容器。答案很接近了，但是不完全正确。

**事实上它是一用来保存字符串对象引用的容器**。

即使字符串是不可变的，它仍然和Java中的其他对象一样。**对象都是创建在堆中，字符串也不例外**。所以字符串常量池仍然依靠堆，他们存储的只是堆中字符串的引用。

目前还没有解释这个池到底是什么，或者它为何存在。好吧，因为字符串对象是不可变的，所以复制多个引用来“共享”这个字符串是安全的。下面来看一个例子：

    public class ImmutableStrings {
            public static void main(String[] args) {
                    String one = "someString";
                    String two = "someString";
                    System.out.println(one.equals(two));
                    System.out.println(one == two);
            }
    }
    // Output
    true
    true

在这个例子中，实在没有必要为一个相同的字符串对象创建两个实例。如果字符串像`StringBuffer`一样是可变的，那么我们会被迫创建两个对象（如果不这样做的话，通过一个引用改变它的值，将会导致其他引用的值也同样改变，从而可能发生错误）。但是，我们知道字符串对象是不能被改变的，我们可以安全地通过两个引用`one`和`two`来使用一个字符串对象。  

这个工作是通过字符串常量池完成的，下面来看一下它是如何完成的：

当一个 **.java** 文件被编译成 **.class** 文件时，和所有其他常量一样，每个字符串字面量都通过一种特殊的方式被记录下来。当一个.class文件被加载时（注意加载发生在初始化之前），JVM在.class文件中寻找字符串字面量。当找到一个时，JVM会检查是否有相等的字符串在常量池中存放了堆中引用。如果找不到，就会在堆中创建一个对象，然后将它的引用存放在池中的一个常量表中。一旦一个字符串对象的引用在常量池中被创建，这个字符串在程序中的所有字面量引用都会被常量池中已经存在的那个引用代替。

所以，在上面的例子中字符串常量池中只有一个引用，就是`someString`这个字符串对象的引用。局部变量`one`和`two`都被赋予了同一个字符串对象的引用。可以通过程序的输出来验证。

`equals`方法检查的是两个字符串对象是否包含相同的数据(`someString`)，而`==`操作符作用在对象上比较的是引用是否相同，这意味着只有两个引用指向的是同一个对象才会返回`true`。所以例子中的两个引用是相等的。从输出可以看到，局部变量`one`和`two`不仅包含相同的数据，而且还指向相同的对象。

来看一下他们之间的关系：

![](/img/in-post/2017-11-02-literal-string-in-java/2.jpg)

注意，对于字符串字面量有一点比较特殊。通过“new”关键字构建时一种不同的方式。下面举一个例子：

    public class ImmutableStrings {
            public static void main(String[] args) {
                    String one = "someString";
                    String two = new String("someString");
                    System.out.println(one.equals(two));
                    System.out.println(one == two);
            }
    }
    // Output
    true
    false

在这个例子中，可以看到由于关键字`new`，最后的结果有一点不同。

此例中，两个字符串字面量仍然被放进了常量池的常量表中，但是当使用`new`时，JVM就会在 **运行时** 创建一个新对象，而不是使用常量表中的引用。虽然例子中的两个字符串引用所指向的对象包含相同的数据`someString`，但是这两个对象并不相同。这一点可以从输出看出来，`equals`方法返回了`true`，而检查引用是否相等的`==`返回`false`。

这表明两个变量指向的是两个不同的字符串对象。

如果你想看图形化的表示，下面就是。要记住引用到常量池的字符串对象是在类加载的时候创建的，而另一个对象是在运行时，当`new String`语句被执行时。

![](/img/in-post/2017-11-02-literal-string-in-java/3.jpg)

如果你想得到两个引用到相同对象的局部变量，你可以使用String类中的定义的`intern()`方法。调用`two.intern()`后，会在字符串常量池中寻找是否有值相等的对象引用。如果有的话，就会返回这个引用，然后你可以把它赋给局部变量。

如果这么做的化，局部变量one和two都是同一个对象的引用，并且在字符串常量池中也存有一个引用，就如同第一张图那样。这时，在运行时创建的第二个字符串对象将会被GC回收。

### 垃圾回收

什么条件下对象才会被垃圾回收？当一个对象不再有引用指向它时，这个对象就会被回收。  
有人注意到字符串字面量在垃圾回收时有什么特殊的地方吗？  
让我们来看一个例子，然后你就会明白。

    public class ImmutableStrings {
            public static void main(String[] args) {
                    String one = "someString";
                    String two = new String("someString");
                    one = two = null;
            }
    }

在主函数结束前，有多少个对象可以被回收？0个？1个？还是2个？

答案是一个。不像大多数对象，字符串字面量总是有一个来自字符串常量池的引用。这就意味着它们会一直有一个引用，所以它们不会被垃圾回收。见下图：

![](/img/in-post/2017-11-02-literal-string-in-java/4.jpg)

如你所见，局部变量`one`和`two`没有指向字符串对象，**但是仍然有一个字符串常量池的引用**。  
所以GC并不会回收这个对象。并且这个对象可以通过之前提到的`intern()`方法访问。

### 总结

对于字符串字面量，下面的几条结论你可以记住。

* 相等的字符串字面量将会指向相同的字符串对象（甚至是在不同包的不同类中）。
* 总之，字符串字面量不会被垃圾回收。绝对不会。
* 在运行时创建的字符串和由字符串字面量创建的是两个不同的对象。
* 对于运行时创建的字符串你可以通过`intern()`方法来重用字符串字面量
* 使用`equals()`方法是比较两个字符串是否相等的最好方式。

---

> 原文标题： **Strings, Literally**  
原作者： **Corey McGlone**  
原文链接： https://javaranch.com/journal/200409/ScjpTipLine-StringsLiterally.html
