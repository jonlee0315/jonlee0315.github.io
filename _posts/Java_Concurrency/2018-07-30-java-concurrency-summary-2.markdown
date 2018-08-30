---
layout:     post
title:      "Java并发知识点总结（二）"
subtitle:   "ReentrantLock, ThreadLocal, 同步容器"
date:       2018-07-30
author:     "Jon Lee"
header-img: "img/in-post/2018-07-27-java-concurrency-summary/bg2.jpg"
catalog: true
categories : Java并发
tags:
    - Java
    - 并发
---

### ReentrantLock

`ReentrantLock`可以替代`synchronized`，先使用`synchronized`实现一段程序。

    public class ReentrantLock1 {
    	synchronized void m1() {
    		System.out.println("m1...");
    		for (int i = 0; i < 10; i++) {
    			try {
    				TimeUnit.SECONDS.sleep(1);
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			}
    			System.out.println(i);
    		}
    	}

    	synchronized void m2() {
    		System.out.println("m2...");
    	}

    	public static void main(String[] args) {
    		ReentrantLock1 rl = new ReentrantLock1();
    		new Thread(() -> rl.m1()).start();
    		try {
    			TimeUnit.SECONDS.sleep(1);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		new Thread(() -> rl.m2()).start();
    	}
    }

用`ReentrantLock`实现上面的代码。

    public class ReentrantLock2 {
    	Lock lock = new ReentrantLock();

    	void m1() {
    		lock.lock(); // synchronized (this)
    		try {
    			System.out.println("m1...");
    			for (int i = 0; i < 10; i++) {
    				TimeUnit.SECONDS.sleep(1);
    				System.out.println(i);
    			}
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		} finally {
    			lock.unlock();
    		}
    	}

    	void m2() {
    		lock.lock();
    		System.out.println("m2...");
    		lock.unlock();
    	}

    	public static void main(String[] args) {
    		ReentrantLock2 rl = new ReentrantLock2();
    		new Thread(() -> rl.m1()).start();
    		try {
    			TimeUnit.SECONDS.sleep(1);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		new Thread(() -> rl.m2()).start();
    	}
    }

*Output*
>m1...  
0  
1  
2  
3  
4  
5  
6  
7  
8  
9  
m2...

`ReentrantLock`可以完成同样的功能，但是必须要手动释放锁。  
使用`synchronized`锁定的话如果遇到异常，会自动释放锁，`lock`必须手动释放锁，所以经常在`finally`中进行锁的释放。

`ReentrantLock`可以进行 **尝试锁定**，使用`tryLock`方法，如果拿不到锁，或者指定时间段内都无法锁定，线程可以选择等待还是继续执行。可以根据`tryLock`的方法的返回值判断是否锁定。这点与`synchronized`不同，如果拿不到锁，`synchronized`会一直等待。所以`ReentrantLock`要更加灵活一些。  
`tryLock`指定时间时，由于要抛出异常，所以在`finally`中要做`unlock`处理。

    public class ReentrantLock3 {
    	Lock lock = new ReentrantLock();

    	void m1() {
    		// same as ReentrantLock2
    	}

    	void m2() {
    		boolean locked = false;
    		try {
    			locked = lock.tryLock(5, TimeUnit.SECONDS);
    			System.out.println("m2 lock : " + locked);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		} finally {
    			if (locked) {
    				lock.unlock();
    			}
    		}
    	}
    }

使用`ReentrantLock`还可以调用`lockInterruptibly`方法，可以对线程的`interrupt`方法做出响应。  
在一个线程等待锁的过程中可以被打断。

    public class ReentrantLock4 {

    	public static void main(String[] args) {
    		Lock lock = new ReentrantLock();

    		Thread t1 = new Thread(() -> {
    			lock.lock();
    			System.out.println("t1 start");
    			try {
    				TimeUnit.SECONDS.sleep(Integer.MAX_VALUE);
    				System.out.println("t1 end");
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			} finally {
    				lock.unlock();
    			}
    		});
    		t1.start();
    		Thread t2 = new Thread(() -> {
    			try {
    				lock.lockInterruptibly(); // 可以对interrupt()方法做出响应
    				System.out.println("t2 start");
    				TimeUnit.SECONDS.sleep(1);
    				System.out.println("t2 end");
    			} catch (InterruptedException e) {
    				System.out.println("Interrupted !");
    			} finally {
    				if (lock.tryLock()) {
    					lock.unlock();
    				}
    			}
    		});
    		t2.start();

    		try {
    			TimeUnit.SECONDS.sleep(10); // 10秒钟之后，主线程打断t2
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		t2.interrupt();
    	}
    }

*Output*
>t1 start  
Interrupted !

由于 **t1** 线程无限地睡下去，所以 **t2** 线程永远也得不到这把锁，那么就可以在主线程中调用`interrupt`方法打断 **t2** 线程。  

`ReentrantLock`可以指定为公平锁。

    public class ReentrantLock5 implements Runnable {

    	private static final ReentrantLock lock = new ReentrantLock(true); // 参数为true，公平锁

    	@Override
    	public void run() {
    		for (int i = 0; i < 100; i++) {
    			lock.lock();
    			System.out.println(Thread.currentThread().getName() + " 获得锁");
    			lock.unlock();
    		}
    	}

    	public static void main(String[] args) {
    		ReentrantLock5 rl = new ReentrantLock5();
    		Thread t1 = new Thread(rl);
    		Thread t2 = new Thread(rl);
    		t1.start();
    		t2.start();
    	}
    }

*Output*
>Thread-0 获得锁  
Thread-1 获得锁  
Thread-0 获得锁  
Thread-1 获得锁  
Thread-0 获得锁  
Thread-1 获得锁  
Thread-0 获得锁  
Thread-1 获得锁  
...

### 同步容器

实现一个固定容量的同步容器，拥有`put`和`get`方法，能够支持2个生产者线程及10个消费者线程的阻塞调用。

    public class MyContainer1<T> {
    	private final LinkedList<T> list = new LinkedList<>();
    	private final int MAX = 10;

    	private synchronized void put(T t) {
    		while (list.size() == MAX) { // 不能用if代替while
    			try {
    				this.wait();
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			}
    		}
    		list.add(t);
    		this.notifyAll(); // 不能用notify
    	}

    	private synchronized T get() {
    		T t = null;
    		while (list.size() == 0) {
    			try {
    				this.wait();
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			}
    		}
    		t = list.removeFirst();
    		this.notifyAll();
    		return t;
    	}

    	public static void main(String[] args) {
    		MyContainer1<String> c = new MyContainer1<>();
    		// 启动消费者线程
    		for (int i = 0; i < 10; i++) {
    			new Thread(() -> {
    				for (int j = 0; j < 3; j++) {
    					System.out.println(Thread.currentThread().getName() + " consume " + c.get());
    				}
    			}, "c" + i).start();
    		}
    		// 启动生产者线程
    		for (int i = 0; i < 2; i++) {
    			new Thread(() -> {
    				for (int j = 0; j < 15; j++) {
    					c.put(Thread.currentThread().getName() + " " + j);
    				}
    			}, "p" + i).start();
    		}
    	}
    }

*Output*
>c9 consume p0 0  
c0 consume p0 9  
c1 consume p0 8  
c1 consume p1 2  
c1 consume p1 3  
...  
c5 consume p0 13  
c2 consume p1 8  
c3 consume p1 5  
c0 consume p1 1  
c0 consume p1 13  
c9 consume p1 0  
c9 consume p1 14  
c2 consume p0 14

实现的容器`MyContainer`，能最大容纳10个元素，主函数中创建了10个消费者，每个消费者执行3次`get`
操作，创建两个生产者，每个生产者生产15个元素，也就是调用15次`put`方法。

有几个地方需要注意，第一就是为什么判断`list.size()`时一定要用`while`循环。  
当容器中的元素个数达到最大时，当前线程会执行`wait`方法释放锁，如果当前有多个生产者线程处于wait状态，此时一个消费者取走了一个元素，然后`notifyAll`唤醒了所有处于等待的生产者线程，那么就会出现一种状况，多个生产者线程不会再判断容器中的元素数量，而是接着`wait`方法执行，向容器内添加元素，这时就会出现容器内的元素数量超过了最大值。

![](/img/in-post/2018-07-27-java-concurrency-summary/2.jpg)

通过上面的图可以更好的理解这一过程。  

第二点需要注意的就是一定要使用`notifyAll`方法，而不是`notify`。  
因为`notify`只能唤醒一个线程，当集合满了时，一个生产者线程唤醒了另一个生产者线程，那么这两个线程将一直等待下去。

使用`Condition`搭配`Lock`实现。

    public class MyContainer2<T> {
    	private final LinkedList<T> list = new LinkedList<>();
    	private final int MAX = 10;
    	private Lock lock = new ReentrantLock();
    	private Condition producer = lock.newCondition();
    	private Condition consumer = lock.newCondition();

    	private void put(T t) {
    		lock.lock();
    		try {
    			while (list.size() == MAX) {
    				producer.await();
    			}
    			list.add(t);
    			consumer.signalAll();
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		} finally {
    			lock.unlock();
    		}
    	}

    	private T get() {
    		T t = null;
    		lock.lock();
    		try {
    			while (list.size() == 0) {
    				consumer.await();
    			}
    			t = list.removeFirst();
    			producer.signalAll();
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		} finally {
    			lock.unlock();
    		}
    		return t;
    	}
    }

### ThreadLocal

`ThreadLocal`是一种自动化机制，可以为使用变量的每个线程都创建不同的存储。根除对变量的共享，以解决任务在共享资源上可能产生冲突的问题。  
下面的代码创建了两个线程，第二个线程更改了`name`的值，由于`Person`是可见的，所以第一个线程会得到新的值。

    public class ThreadLocal1 {
    	private volatile static Person p = new Person();

    	public static void main(String[] args) {
    		new Thread(() -> {
    			try {
    				TimeUnit.SECONDS.sleep(2);
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			}
    			System.out.println(p.name);
    		}).start();

    		new Thread(() -> {
    			try {
    				TimeUnit.SECONDS.sleep(1);
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			}
    			p.name = "James";
    		}).start();
    	}
    }

    class Person {
    	String name = "Jon";
    }

如果现在想线程之间互不影响，我在一个线程中改变属性的值，其他线程不会改变。  
使用`ThreadLocal`，为每个线程设置局部变量。

public class ThreadLocal2 {
	static ThreadLocal<Person> tl = new ThreadLocal<>();

	public static void main(String[] args) {
		new Thread(() -> {
			try {
				TimeUnit.SECONDS.sleep(2);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread().getName() + " person = " + tl.get());
		}, "t1").start();

		new Thread(() -> {
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			tl.set(new Person());
			System.out.println(Thread.currentThread().getName() + " person = " + tl.get().name);
		}, "t2").start();
	}
}

*Output*
>t2 person = Jon  
t1 person = null

`ThreadLocal`是一种空间换时间的方法，而`synchronized`是时间换空间。

### 线程安全的Singleton

单例模式有多种实现方式，下面列出两种常用的线程安全实现。

* 双重校验锁（Double Check lock）

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

* 静态内部类

        public class Singleton {

        	private Singleton() {
        	}

        	public static Singleton getInstance() {
        		return SingletonHolder.instance;
        	}

        	private static class SingletonHolder {
        		private static final Singleton instance = new Singleton();
        	}
        }

单例模式的详细分析，包括多种单例模式的介绍，可以参考这篇博客[设计模式之Singleton模式](http://lijiaheng.me/设计模式/2018/05/04/design-pattern-singleton)。

### 参考资料

>https://www.youtube.com/watch?v=_D7MiAEM3oY&list=PL0onFhfJfEDD2KcgyXlAFg20XDLabnXqz

---
