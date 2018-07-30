---
layout:     post
title:      "Java8 新特性总结"
subtitle:   "Java 8 New Features"
date:       2017-10-23
author:     "Jon Lee"
header-img: "img/in-post/2017-10-23-java-8-new-features/bg.jpg"
catalog:    true
categories : Java基础
tags:
    - Java
---

### 接口中的默认方法和静态方法

先考虑一个问题，如何向Java中的集合库中增加方法？  
例如，在Java 8中向`Collection`接口中添加了一个`forEach`方法。

如果在Java 8之前，对于接口来说，其中的方法必须都为抽象方法，也就是说接口中不允许有接口的实现，那么就需要对每个实现`Collection`接口的类都需要实现一个`forEach`方法。

但这就会造成在给接口添加新方法的同时影响了已有的实现，所以Java设计人员引入了接口默认方法，其目的是为了解决接口的修改与已有的实现不兼容的问题，接口默认方法可以作为库、框架向前兼容的一种手段。

默认方法就像一个普通Java方法，只是方法用`default`关键字修饰。

下面来举一个简单的例子:
定义`Move`接口，在其中实现默认方法`start()`。

    public interface Move {

    	default void start(String name) {
    		System.out.println(name + " start moving");
    	}
    }

创建一个`Tank`类实现`Move`接口。

    public class Tank implements Move {
        // blank
    }

创建实例，调用`start方法。`

    public static void main(String[] args) {
    	Tank tank = new Tank();
    	tank.start("tank");
    }

可以从结果中看到，**虽然Tank类中没有实现任何方法，但是仍然可以实现getName()方法**。

显然默认接口的出现打破了之前的一些基本规则，使用时要注意几个问题。  
如果一个接口中定义了一个默认方法，而另外一个父类或者接口中又定义了一个同名的方法，该选择哪个？

1. 选择父类中的方法。如果一个父类提供了具体的实现方法，那么接口中具有相同名称和参数的默认方法会被忽略。也就是遵守 **“类优先”原则**，即当类和接口都有一个同名方法时，只有父类中的方法会起作用。“类优先”原则可以保证与Java 7的兼容性。如果你再接口中添加了一个默认方法，它对Java 8以前编写的代码不会产生任何影响。

2. 接口冲突。如果一个父接口提供了一个默认方法，而另一个接口也提供了具有相同名称和参数类型的方法（不管该方法是否是默认方法），那么必须通过覆盖方法来解决。
比如增加一个`Attack`接口，其中也有一个`start`方法。

        public interface Attack {

        	default void start(String name) {
        		System.out.println(name + " start attacking");
        	}
        }

    那么如果一个类同时实现`Attack`和`Move`接口，会出现编译错误。
    ![](/img/in-post/2017-10-23-java-8-new-features/1.jpg)
    只能通过重写其中一个方法来解决。

        public class Tank implements Move, Attack {

        	public static void main(String[] args) {
        		Tank tank = new Tank();
        		tank.start("tank");
        	}

        	@Override
        	public void start(String name) {
        		Attack.super.start(name);
        	}
        }

接口中的静态方法就像一个普通Java静态方法，在方法前加`static`关键字，但方法的权限修饰只能是public或者不写。

在Java 8中`Collection`接口中就添加了四个默认方法，stream()、parallelStream()、spliterator()和removeIf()。之前所说的`forEach()`是在`Collection`的父接口`Iterable`接口中定义的默认方法。`Comparator`接口也增加了许多默认方法和静态方法。

### 函数式接口和Lambda表达式

**函数式接口**（Functional Interface）是只包含一个方法的抽象接口。  
比如Java标准库中的`java.lang.Runnable`，`java.util.concurrent.Callable`就是典型的函数式接口。

在Java 8中通过`@FunctionalInterface`注解，将一个接口标注为函数式接口，该接口只能包含一个抽象方法。  

**@FunctionalInterface注解不是必须的，只要接口只包含一个抽象方法，虚拟机会自动判断该接口为函数式接口。**

一般建议在接口上使用@FunctionalInterface注解进行声明，以免他人错误地往接口中添加新方法，如果在你的接口中定义了第二个抽象方法的话，编译器会报错。

![](/img/in-post/2017-10-23-java-8-new-features/2.jpg)

函数式接口是为Java 8中的lambda而设计的，**lambda表达式的方法体其实就是函数接口的实现**。

为什么要使用lambda表达式？  
lambda表达式是一段可以传递的代码，因为他可以被执行一次或多次。

当我们在一个线程中执行一些逻辑时，通常会将代码放在一个实现Runnable接口的类的run方法中，如下所示：

    new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < 10; i++)
                        System.out.println("Without Lambda Expression");
                }}).start();

然后通过创建实例来启动一个新的线程。run方法内包含了一个新线程中需要执行的代码。  

再来看另一个例子，如果想利用字符串长度排序而不是默认的字典顺序排序，就需要自定义一个实现Comparator接口的类，然后将对象传递给sort方法。

    class LengthComparator implements Comparator<String> {
            @Override
            public int compare(String s1, String s2) {
                    return Integer.compare(s1.length(), s2.length());
            }
    }
    Arrays.sort(strings, new LengthComparator());

按钮回调是另一个例子。将回调操作放在了一个实现了监听器接口的类的一个方法中。

    JButton button = new JButton("click");

    button.addActionListener(new ActionListener() {    
            @Override
            public void actionPerformed(ActionEvent e) {
                    System.out.println("Without Lambda Expression");
            }
    });


这三个例子中，出现了相同方式，一段代码被传递给其他调用者——一个新线程、是一个排序方法或者是一个按钮。这段代码会在稍后被调用。  
在Java中传递代码并不是很容易，不可能将代码块到处传递。你不得不构建一个类的对象，由它的某个方法来包含所需的代码。  

而lambda表达式实际上就是代码块的传递的实现。其语法结构如下：  
`(parameters) -> expression` 或者 `(parameters) -> {statements;}`

括号里的参数可以省略其类型，编译器会根据上下文来推导参数的类型，你也可以显式地指定参数类型，如果没有参数，括号内可以为空。  
方法体，如果有多行功能语句用大括号括起来，如果只有一行功能语句则可以省略大括号。

上面的代码使用lambda表达式简化后：

    // Runnable
    new Thread(() -> {
                for (int i = 0; i < 100; i++)
                    System.out.println("Lambda Expression");
            }).start();
    // Comparator
    Comparator<String> c = (s1, s2) -> Integer.compare(s1.length(), s2.length());
    // ActionListener
    button.addActionListener(e -> System.out.println("Lambda Expression"));

可以看到lambda表达式使代码变得简单，代替了匿名内部类。

下面来说一下方法引用，方法引用是lambda表达式的一种简写形式。  
如果lambda表达式只是调用一个特定的已经存在的方法，则可以使用方法引用。  
使用`::`操作符将方法名和对象或类的名字分隔开来。以下是四种使用情况：

* 对象 :: 实例方法
* 类 :: 静态方法
* 类 :: 实例方法
* 类 :: new  

下面的代码就是第三种情况，对lambda表达式又一次进行了简化。

    Arrays.sort(strings, String::compareToIgnoreCase);
    // 等价于
    Arrays.sort(strings, (s1, s2) -> s1.compareToIgnoreCase(s2));

### Stream API

当处理集合时，通常会迭代所有元素并对其中的每一个进行处理。  
例如，我们希望统计一个字符串类型数组中，所有长度大于3的元素。

    String[] strArr = { "Java8", "new", "feature", "Stream", "API" };
    int count = 0;
    for (String s : strArr) {
            if (s.length() > 3)
                    count++;
    }

通常我们都会使用这段代码来统计，并没有什么错误，只是它很难被并行计算。  
这也是 Java8 引入大量操作符的原因，在 Java8 中，实现相同功能的操作符如下所示：

    long count = Stream.of(strArr).filter(w -> w.length() > 3).count();

`Stream.of`方法会为字符串列表生成一个 **Stream** 对象。  
`filter`方法会返回只包含字符串长度大于3的一个Stream，然后通过`count`方法计数。

一个Stream表面上与一个集合很类似，允许你改变和获取数据，但实际上却有很大区别：

1. Stream自己不会存储元素。元素可能被存储在底层的集合中，或者根据需要产生出来。
2. Stream操作符不会改变源对象。相反，他们返回一个持有新结果的Stream。
3. Stream操作符可能是延迟执行的。意思是它们会等到需要结果的时候才执行。

Stream相对于循环操作有更好的可读性。并且可以并行计算：

    long count = Arrays.asList(strArr).parallelStream().filter(w -> w.length() > 3).count();

只需要把stream方法改成parallelStream，就可以让Stream去并行执行过滤和统计操作。

**Stream** 遵循“做什么，而不是怎么去做”的原则。只需要描述需要做什么，而不用考虑程序是怎样实现的。**Stream** 很像 **Iterator**，单向，只能遍历一遍。但是 **Stream** 可以只通过一行代码就实现多线程的并行计算。

当使用 **Stream** 时，会有三个阶段：
1. 创建一个Stream。
2. 在一个或多个步骤中，将初始Stream转化到另一个Stream的中间操作。
3. 使用一个终止操作来产生一个结果。该操作会强制他之前的延迟操作立即执行。在这之后，该Stream就不会在被使用了。

从着三个阶段来看，对应着三种类型的方法，首先是Stream的 **创建方法**。

    // 1. Individual values
    Stream stream = Stream.of("a", "b", "c");
    // 2. Arrays
    String [] strArray = new String[] {"a", "b", "c"};
    stream = Stream.of(strArray);
    stream = Arrays.stream(strArray);
    // 3. Collections
    List<String> list = Arrays.asList(strArray);
    stream = list.stream();

**中间操作** 包括：map (mapToInt, flatMap 等)、 filter、distinct、sorted、peek、limit、skip、parallel、sequential、unordered。

**终止操作** 包括：forEach、forEachOrdered、toArray、reduce、collect、min、max、count、anyMatch、allMatch、noneMatch、findFirst、findAny、iterator。

### 新的时间和日期类 API

Java8 引入了一个新的日期和时间API，位于`java.time`包下。  
新的日期和时间API借鉴了Joda Time库，其作者也为同一人，但它们并不是完全一样的，做了很多改进。

下面来说一下几个常用的类。首先是`Instant`，一个`Instant`对象表示时间轴上的一个点。  
`Instant.now()`会返回当前的瞬时点（格林威治时间）。`Instant.MIN`和`Instant.MAX`分别为十亿年前和十亿年后。  
如下代码可以计算某算法的运行时间：

    Instant start = Instant.now();
    doSomething();
    Instant end = Instant.now();
    Duration timeElapsed = Duration.between(start, end);
    long millis = timeElapsed.toMillis();

`Duration`对象表示两个瞬时点间的时间量。可以通过不同的方法，换算成各种时间单位。

上面所说的绝对时间并不能应用到生活中去，所以新的Java API中提供了两种人类时间，本地日期/时间和带时区的时间。  
`LocalDate`是一个带有年份、月份和天数的日期。创建他可以使用静态方法`now`或者`of`。

    LocalDate today = LocalDate.now();
    LocalDate myBirthday = LocalDate.of(1994, 03, 15);
    // use Enum
    myBirthday = LocalDate.of(1994, Month.MARCH, 15);

    System.out.println(today); // 2017-10-23
    System.out.println(myBirthday); // 1994-03-15

下面是`LocalDate`中的一些常用方法：

![](/img/in-post/2017-10-23-java-8-new-features/3.jpg)

`LocalTime`表示一天中的某个时间，同样可以使用`now`或者`of`来创建实例。

    LocalTime rightNow = LocalTime.now();
    LocalTime bedTime = LocalTime.of(2, 0);
    System.out.println(rightNow); // 01:26:17.139
    System.out.println(bedTime); // 02:00
    LocalDateTime表示一个日期和时间，用法和上面类似。

上面几种日期时间类都属于本地时间，下面来说一下带时区的时间。
`ZonedDateTime`通过设置时区的id来创建一个带时区的时间。

    ZonedDateTime beijingOlympicOpenning = ZonedDateTime.of(2008, 8, 8, 20, 0, 0, 0, ZoneId.of("Asia/Shanghai"));
    System.out.println(beijingOlympicOpenning); // 2008-08-08T20:00+08:00[Asia/Shanghai]

更新后的API同样加入了新的格式化类`DateTimeFormatter`。  
`DateTimeFormatter`提供了三种格式化方法来打印日期/时间：

* 预定义的标准格式

        String formattered = DateTimeFormatter.ISO_LOCAL_DATE_TIME.format(beijingOlympicOpenning);
        System.out.println(formattered); // 2008-08-08T20:00:00

    `DateTimeFormatter`类提供了多种预定义的标准格式可供使用。

* 语言环境相关的格式

    标准格式主要用于机器可读的时间戳。为了让人能够读懂日期和时间，你需要使用语言环境相关的格式。  
    Java8提供了4种风格，`SHORT`、`MEDIUM`、`LONG`、`FULL`。

        String formattered = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.FULL).format(beijingOlympicOpenning);
        System.out.println(formattered);    //2008年8月8日 星期五 下午08时00分00秒 CST

* 自定义的格式

    你也可以自定义日期和时间的格式。

        String formattered = DateTimeFormatter.ofPattern("E yyyy-MM-dd HH:mm").format(beijingOlympicOpenning);
        System.out.println(formattered); // 星期五 2008-08-08 20:00

新的API提供了从字符串解析出日期/时间的`parse`静态方法和与遗留类（`java.util.Date`、`java.sql.Time`和`java.txt.DateFormat`等）互相转换的方法。

### 杂项改进

* Java8在`String`类中只添加了一个新方法，就是`join`，该方法实现了字符串的拼接，可以把它看作`split`方法的逆操作。

        String joined = String.join(".", "www", "lijiaheng", "me");
        System.out.println(joined); // www.lijiaheng.me

* 数字包装类提供了`BYTES`静态方法，以 **byte** 为单位返回长度。

* 所有八种包装类都提供了静态的`hashCode`方法。

* **Short**、**Integer**、**Long**、**Float** 和 **Double** 这5种类型分别提供了了`sum`、`max`和`min`，用来在流操作中作为聚合函数使用。

* 集合类和接口中添加的方法：

    ![](/img/in-post/2017-10-23-java-8-new-features/4.jpg)

* Java8为使用流读取文件行及访问目录项提供了一些简便的方法（`Files.lines`和`Files.list`）。同时也提供了进行Base64编码/解码的方法。

* Java8对GUI编程（JavaFX）、并发等方面也做了改进，目前还没看，所以只总结到这里。

### 参考资料
>《写给大忙人看的Java SE 8》  
https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/
