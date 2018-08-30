---
layout:     post
title:      "设计模式之单例模式"
subtitle:   "Singleton —— “只有一个实例”"
date:       2018-05-04
author:     "Jon Lee"
header-img: "img/in-post/design-pattern/singleton_bg.jpg"
catalog: true
categories : 设计模式
tags:
    - Java
    - 设计模式
---

### Singleton模式

单例模式是一个创建型模式，目的是确保一个类只有一个实例，并提供对该实例的全局访问。  
单例模式包含的类只有一个，就是`Singleton`类。  
`Singleton`中必须保证用户无法通过`new`关键字直接实例化它，所以构造函数应为`private`。  
并且包含一个静态私有成员变量和一个静态公有的工厂方法，工厂方法负责检验实例是否存在并创建实例。

### UML类图

![](/img/in-post/design-pattern/singleton_1.png)

### 饿汉式

所谓饿汉式就是只要类被加载时就创建实例，而不管你此时用没用到它。   

    public class Singleton {
    	private static Singleton instance = new Singleton();

    	private Singleton() {
    	}

    	public static Singleton getInstance() {
    		return instance;
    	}
    }

`java.lang.Runtime`就是使用的这种方式：

![](/img/in-post/design-pattern/singleton_2.png)

这种方法的缺点就是如果实例一直没有用到，并且占用内存比较大时，就浪费了资源。   
如果遇到这种问题就要用下面的方法，使用懒加载的方式。

### 懒汉式（非线程安全）

懒汉式，只有外部类用到`Singleton`时才会创建实例，解决了饿汉式资源浪费的问题。  
但是这种延迟加载方式并不是线程安全的，在多个线程同时访问`getInstance`方法时，由于方法没有加锁，可能会出现实例被创建多次的问题，这也就打破了单例模式的原则。

    public class Singleton {
    	private static Singleton instance;

    	private Singleton() {
    	}

    	public static Singleton getInstance() {
    		if (instance == null) {
    			instance = new Singleton();
    		}
    		return instance;
    	}
    }

### 懒汉式（线程安全）

解决懒汉式线程安全问题的最简单方法就是加同步锁，在工厂方法上加`synchronized`关键字。  
但是这种方法也是有缺陷的，由于在方法上加锁，当访问的线程太多时只能阻塞，等待释放锁，效率不高。

    public class Singleton {
    	private static Singleton instance;

    	private Singleton() {
    	}

    	public static synchronized Singleton getInstance() {
    		if (instance == null) {
    			instance = new Singleton();
    		}
    		return instance;
    	}
    }

### 双重校验锁

在学习`synchronized`时，就有一条原则，只在需要的地方加锁。  
所以如果要改进上面方法的话，我们只需要在判断是否为`null`和创建实例的地方加锁就好。
这就是下面的方法，双重校验锁（Double-Checked Locking）。  
使用`volatile`确保实例的可见性，并且禁止重排序，工厂方法中第一个判断是否为`null`减少了不必要的同步。

    public class Singleton {
    	private volatile static Singleton instance;

    	private Singleton() {
    	}

    	private static Singleton getInstance() {
    		if (instance == null) {
    			synchronized (Singleton.class) {
    				if (instance == null) {
    					instance = new Singleton();
    				}
    			}
    		}
    		return instance;
    	}
    }

### 静态内部类

使用静态内部类的方法也可以实现懒加载，并且是线程安全的。  
只有第一次调用`getInstance`方法时，JVM才会加载内部类`SingletonHolder`，并初始化`instance`。

    public class Singleton {
    	private Singleton() {
    	}

    	private static Singleton getInstance() {
    		return SingletonHolder.instance;
    	}

    	private static class SingletonHolder {
    		private static final Singleton instance = new Singleton();
    	}
    }

### 枚举

《Effective Java》中推荐的一种实现方法，虽然很简洁，但是可读性不强。

    public enum Singleton {
            INSTANCE;

            public void doSomething() {
            }
    }

### 参考资料

>《图解设计模式》  
《Effective Java》  
https://itimetraveler.github.io/2016/09/08/【Java】设计模式：深入理解单例模式/  
http://design-patterns.readthedocs.io/zh_CN/latest/creational_patterns/singleton.html

---
