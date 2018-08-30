---
layout:     post
title:      "设计模式之工厂方法模式"
subtitle:   "Factory Method —— “将实例的生成交给子类”"
date:       2018-05-07
author:     "Jon Lee"
header-img: "img/in-post/design-pattern/factory_bg.jpg"
catalog: true
categories : 设计模式
tags:
    - Java
    - 设计模式
---

### Factory Method模式

在工厂方法模式中，父类决定实例的生成方式，但不决定所要生成的具体的类，具体的处理全部交给子类负责。  
目的是将生成实例的框架和实际负责生成实例的类解耦。

### UML类图

![](/img/in-post/design-pattern/factory_method.png)

从图中可以看出父类（框架）这一方的 **Creator** 角色与 **Product** 角色的关系与子类（具体加工）这一方的 **ConcreteCreator** 角色和 **ConcreteProduct** 角色的关系是平行的。

**Creator** 类是负责生产 **Product** 的抽象类，具体实现是由 **ConcreteCreator** 负责。  
**Creator** 并不知道具体生产的是什么，只需要通过 **ConcreteCreator** 调用`factoryMethod`就可以创建 **Product** 实例。  
方法中不需要用`new`关键字来创建实例，而是调用生成实例的专用方法，防止父类与其他具体类耦合。

### 小例子

假设我们现在要构建一个能创造小动物的工厂，使用工厂方法模式进行解耦。

创建一个动物工厂的抽象类`Creator`：

    public abstract class AnimalFactory {
    	protected abstract Animal createAnimal(String name);
    }

创建动物抽象类`Product`，里面有一个`eat`方法：

    public abstract class Animal {
    	public abstract void eat();
    }

创建两个实体工厂类`ConcreteCreator`，分别生产小猫和小狗：

    public class CatFactory extends AnimalFactory {
    	@Override
    	public Animal createAnimal(String name) {
    		return new Cat(name);
    	}
    }

    public class DogFactory extends AnimalFactory {
    	@Override
    	public Animal createAnimal(String name) {
    		return new Dog(name);
    	}
    }

最后是`ConcreteProduct`，小猫和小狗的实体类：

    public class Cat extends Animal {
    	private String name;

    	Cat(String name) {
    		this.name = name;
    		System.out.println("制作小猫 " + name);
    	}

    	@Override
    	public void eat() {
    		System.out.println("小猫 " + name + " 吃鱼");
    	}
    }

    public class Dog extends Animal {
    	private String name;

    	Dog(String name) {
    		this.name = name;
    		System.out.println("制造小狗 " + name);
    	}

    	@Override
    	public void eat() {
    		System.out.println("小狗 " + name + " 吃骨头");
    	}
    }

测试类：

    public class Main {
    	public static void main(String[] args) {
    		AnimalFactory catFactory = new CatFactory();
    		AnimalFactory dogFactory = new DogFactory();
    		Animal a1 = catFactory.createAnimal("小花");
    		Animal a2 = catFactory.createAnimal("小红");
    		Animal a3 = dogFactory.createAnimal("旺旺");
    		Animal a4 = dogFactory.createAnimal("汪汪");
    		System.out.println("==========开饭了==========");
    		a1.eat();
    		a2.eat();
    		a3.eat();
    		a4.eat();
    	}
    }

*Output*
>制作小猫 小花  
制作小猫 小红  
制造小狗 旺旺  
制造小狗 汪汪  
==========开饭了==========  
小猫 小花 吃鱼  
小猫 小红 吃鱼  
小狗 旺旺 吃骨头  
小狗 汪汪 吃骨头

### 参考资料

> 《图解设计模式》
