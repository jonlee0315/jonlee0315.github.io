---
layout:     post
title:      "Java并发知识点总结（一）"
subtitle:   "Java Concurrency Summary"
date:       2018-07-27
author:     "Jon Lee"
header-img: "img/in-post/2018-07-27-java-concurrency-summary/bg.jpg"
catalog: true
categories : Java并发
tags:
    - Java
    - 并发
---

### synchronized关键字
---
 1. `synchronized`是一种互斥锁，在给定时刻只允许一个任务访问资源。  
使用`synchronized`对某个对象加锁，只有拿到当前对象的锁的线程才能够执行其中的方法。

        public class TestSynchronized1 {
        	private int count = 10;
        	private Object o = new Object();

        	public void m() {
        		synchronized (o) {
        			count--;
        			System.out.println(Thread.currentThread().getName() + " count = " + count);
        		}
        	}
        }

2. 使用`this`作为被锁定对象，避免创建不必要的对象。

        public class TestSynchronized2 {
        	private int count = 10;

        	public void m() {
        		// 避免创建o对象
        		synchronized (this) {
        			count--;
        			System.out.println(Thread.currentThread().getName() + " count = " + count);
        		}
        	}
        }

3. 如果`synchronized`作用在方法上，同于在方法中的代码外加上`synchronized (this)`，下面的代码等同于上一个例子。

        public class TestSynchronized3 {
        	private int count = 10;

        	public synchronized void m() {
        		count--;
        		System.out.println(Thread.currentThread().getName() + " count = " + count);
        	}
        }

4. 在静态方法上使用`synchronized`，因为`static`方法中没有`this`，所以上锁的对象是`Class`对象。

        public class TestSynchronized4 {
        	private static int count = 10;

        	public synchronized static void m1() { // 等同于synchronized(TestSynchronized4.class)
        		count--;
        		System.out.println(Thread.currentThread().getName() + " count = " + count);
        	}

        	public static void m2() {
        		synchronized (TestSynchronized4.class) {
        			count--;
        		}
        	}
        }

5. 同步方法和非同步方法可以同时调用

        public class TestSynchronized5 {
        	synchronized void m1() {
        		System.out.println(Thread.currentThread().getName() + " m1 start...");
        		try {
        			Thread.sleep(10000);
        		} catch (InterruptedException e) {
        			e.printStackTrace();
        		}
        		System.out.println(Thread.currentThread().getName() + " m1 end");
        	}

        	void m2() {
        		System.out.println(Thread.currentThread().getName() + " m2 start...");
        		try {
        			Thread.sleep(5000);
        		} catch (InterruptedException e) {
        			e.printStackTrace();
        		}
        		System.out.println(Thread.currentThread().getName() + " m2 end");
        	}

        	public static void main(String[] args) {
        		TestSynchronized5 t = new TestSynchronized5();
        		new Thread(() -> t.m1()).start();
        		new Thread(() -> t.m2()).start();
        	}
        }

    *Output*

    >Thread-0 m1 start...  
    Thread-1 m2 start...  
    Thread-1 m2 end  
    Thread-0 m1 end

6. 在一个同步方法中可以调用另外一个同步方法，由于`synchronized`是可重入锁，当前线程已经拥有某个对象的锁，再次申请的时候仍然会得到该对象的锁。

        public class TestSynchronized6 {
        	synchronized void m1() {
        		System.out.println("m1 start");
        		m2();
        	}

        	synchronized void m2() {
        		System.out.println("m2 start");
        	}

        	public static void main(String[] args) {
        		TestSynchronized6 t = new TestSynchronized6();
        		new Thread(() -> t.m1()).start();
        	}
        }

7. 同样，因为`synchronized`是可重入锁，在子类可以调父类同步方法，避免了死锁的产生。

        public class TestSynchronized7 {
        	protected synchronized void hello() {
        		System.out.println("parent begin");
        		try {
        			TimeUnit.SECONDS.sleep(5);
        		} catch (InterruptedException e) {
        			e.printStackTrace();
        		}
        		System.out.println("parent end");
        	}

        	public static void main(String[] args) {
        		new Child().hello();
        	}
        }

        class Child extends TestSynchronized7 {
        	@Override
        	protected synchronized void hello() {
        		System.out.println("child begin");
        		super.hello();
        		System.out.println("child end");
        	}
        }

8. 使用`synchronized`的方法如果出现异常，会自动释放锁。

        public class TestSynchronized8 {
        	int count = 0;

        	synchronized void m() {
        		System.out.println(Thread.currentThread().getName() + " start");
        		while (true) {
        			count++;
        			System.out.println(Thread.currentThread() + " count = " + count);
        			try {
        				TimeUnit.SECONDS.sleep(1);
        			} catch (InterruptedException e) {
        				e.printStackTrace();
        			}
        			if (count == 5) {
        				int i = 1 / 0; // 手动捕获异常不会释放锁，t1继续运行
        			}
        		}
        	}

        	public static void main(String[] args) {
        		TestSynchronized8 t = new TestSynchronized8();
        		new Thread(() -> t.m(), "t1").start();
        		try {
        			TimeUnit.SECONDS.sleep(1);
        		} catch (InterruptedException e) {
        			e.printStackTrace();
        		}
        		new Thread(() -> t.m(), "t2").start();
        	}
        }

    t2拿到锁继续执行  
    *Output*
    >t1 start  
    Thread[t1,5,main] count = 1  
    Thread[t1,5,main] count = 2  
    Thread[t1,5,main] count = 3  
    Thread[t1,5,main] count = 4  
    Thread[t1,5,main] count = 5  
    t2 start
    Exception in thread "t1" java.lang.ArithmeticException: / by zero  
    Thread[t2,5,main] count = 6  
    Thread[t2,5,main] count = 7  
    Thread[t2,5,main] count = 8
    ...

9. 同步代码块中的语句越少越好

        public TestSynchronized9 {
        	int count = 0;

        	synchronized void m1() {
        		// do something
        		count++;
        		// do something
        	}

        	void m2() {
        		// do something
        		// 业务逻辑中只有下面这句需要同步，不应该给整个方法上锁，采用细粒度的锁可以使线程争用时间变短，从而提高效率
        		synchronized (this) {
        			count++;
        		}
        		// do something
        	}
        }

10. 避免用字符串做对象锁，下面的例子中s1 == s2。  
如果使用的类库中使用了"Hello"作为锁，由于你无法读到源码，所以会出现死锁，因为不经意间使用了同一把锁。

        public class TestSynchronized10 {
        	String s1 = "Hello";
        	String s2 = "Hello";

        	void m1() {
        		synchronized (s1) {
        		}
        	}
        	void m2() {
        		synchronized (s2) {
        		}
        	}
        }

### 脏读
---
来看一下下面的代码

    public class Account {
    	private String name;
    	private double balance;

    	public static void main(String[] args) {
    		Account a = new Account();
    		new Thread(() -> a.set("account1", 250000)).start();
    		try {
    			TimeUnit.SECONDS.sleep(1);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		System.out.println(a.getBalance());
    		try {
    			TimeUnit.SECONDS.sleep(1);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		System.out.println(a.getBalance());
    	}

    	public synchronized void set(String name, double balance) {
    		this.name = name;
    		try {
    			TimeUnit.SECONDS.sleep(2);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		this.balance = balance;
    	}

    	public double getBalance() {
    		return this.balance;
    	}
    }

*Output*
>0.0  
250000.0

分析一下这段程序，首先开启一个线程调用`set()`方法，对对象属性进行赋值，假设在这过程中有一段操作，我们模拟`sleep`2秒，而这时主线程调用了`getBalance()`方法，此时由于子线程还在阻塞中，所以`balance`属性还没有来得及被赋值。所以产生这种情况的主要原因是
**我们只对写操作加了锁，但是并没有对读操作加锁**，所以解决这个问题只需要在读方法加上`synchronized`即可，只有写操作完成后释放锁，读操作才能运行。

    public synchronized double getBalance() {
            return this.balance;
    }

*Output*
>250000.0  
250000.0

### volatile关键字
---
`votalile`是Java中最轻量级的同步机制，要理解`volatile`可以先了解一下Java的内存模型。  
先来看一下程序，成员变量`running`的初值为**true**，启动一个新线程**t1**执行`m()`方法，在方法中有一个死循环，条件是`running`为true，主线程在睡了1秒钟之后将`running`变量的值改为false，那么t1线程是否会停止呢？

    public class TestVolatile1 {
    	boolean running = true;

    	void m() {
    		System.out.println("m start");
    		while (running) {
    		}
    		System.out.println("m end");
    	}

    	public static void main(String[] args) {
    		TestVolatile1 t = new TestVolatile1();
    		new Thread(() -> t.m(), "t1").start();
    		try {
    			TimeUnit.SECONDS.sleep(1);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		t.running = false;
    	}
    }

答案是并不会，首先要弄清楚这个问题需要先知道Java的 **内存模型** 到底是什么。
![](/img/in-post/2018-07-27-java-concurrency-summary/1.jpg)
Java内存模型的主要目标是定义程序中各个变量的访问规则，即在虚拟机中将变存储到内存和从内存中读取到变量这样的底层细节。（这里的变量指的是成员变量、静态变量等线程间共享的数据）  

Java内存模型规定了所有的变量都存储在**主内存**（Main Memory）中，每条线程有着自己的工作内存，线程的工作内存中保存了被该线程使用到的变量的主内存副本拷贝。线程对变量的操作都必须在自己的工作内存中进行，它们之间的交互方式如上图。

再来看之前的代码，线程 **t1** 启动后，它的工作内存中`running`的值一直为true，因为即使主线程将`running`设置为 **false** 后将这个变量值同步回主存，线程**t1**也不会知道，所以它会一直执行下去。  

如果将`while`循环中加入一些内容，**t1** 就可能会停下来，因为当空闲时t1可能会去主存中刷新一下变量的值。

    while (running) {
            System.out.println("running");
    }

那么怎么保证当一个线程修改了一个变量的值，其他线程都会立即得知呢？  
使用`volatile`关键字，`volatile`可以保证“可见性”，并且禁止指令重排序优化。

要注意一点`volatile`不能替代`synchronized`，因为 **`volatile`修饰变量的运算在并发下是不安全的** 。

    public class TestVolatile2 {
    	private volatile int count = 0;

    	public void m() {
    		for (int i = 0; i < 10000; i++) {
    			count++;
    		}
    	}

    	public static void main(String[] args) {
    		List<Thread> list = new ArrayList<>();
    		TestVolatile2 t = new TestVolatile2();
    		for (int i = 0; i < 10; i++) {
    			list.add(new Thread(() -> t.m()));
    		}
    		list.forEach((o) -> o.start());
    		list.forEach((o) -> {
    			try {
    				o.join();
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			}
    		});
    		System.out.println(t.count);
    	}
    }

多运行几次上面的代码，会发现每次的结果都是不同的，并且都小于100000。  
问题出在 **count++** 并不是一个原子操作，即使你对`count`加了`volatile`关键字修饰，但是其只能保证该变量在多个线程之间是可见的，当对`count`做出修改时，同步回主存的值可能还没有+1，所以总数会小于100000。

解决方法，使用`synchronized`加锁，保证`count++`操作在一个线程中运行结束不会被打断。

    public synchronized void m() {
            for (int i = 0; i < 10000; i++) {
                    count++;
            }
    }

或者使用`AtomInteger`原子类。

        private AtomicInteger count = new AtomicInteger(0);

        public void m() {
                for (int i = 0; i < 10000; i++) {
                        count.incrementAndGet();
                }
        }

### 容器问题
---
>实现一个容器，提供两个方法`add`和`size`
创建两个线程，**线程1** 添加 **10** 个元素到容器中，**线程2** 实时监控容器中的元素数量，当数量等于 **5** 时打印提示。

利用到上面的知识和根据题目要求，当一个线程更改变量的属性时，另一个线程要立刻收到通知，得到当前改变后的值，所以我们要保证变量的 **可见性**， 也就是使用`volatile`关键字，由此可以写出下面的程序。

    public class MyContainer1 {
    	volatile List<Object> list = new ArrayList<>();

    	public void add(Object o) {
    		list.add(o);
    	}

    	public int size() {
    		return list.size();
    	}

    	public static void main(String[] args) {
    		MyContainer1 c = new MyContainer1();
            // t1 线程添加元素
    		new Thread(() -> {
    			for (int i = 0; i < 10; i++) {
    				c.add(new Object());
    				System.out.println("add " + i);
    				try {
    					TimeUnit.SECONDS.sleep(1);
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    		}).start();
            // t2 线程监控
    		new Thread(() -> {
    			while (true) {
    				if (c.size() == 5) {
    					break;
    				}
    			}
    			System.out.println("size is 5");
    		}).start();
    	}
    }

*Output*
>add 0  
add 1  
add 2  
add 3  
add 4  
size is 5  
add 5  
add 6  
add 7  
add 8  
add 9

上面的代码可以达到题目要求，但是 **t2** 线程中的死循环太浪费CPU资源，下面利用`wait`和`notify`来优化。

    public class MyContainer2 {
    	List<Object> list = new ArrayList<>();

    	public void add(Object o) {
    		list.add(o);
    	}

    	public int size() {
    		return list.size();
    	}

    	public static void main(String[] args) {
    		final Object lock = new Object();
    		MyContainer2 c = new MyContainer2();
    		new Thread(() -> {
    			synchronized (lock) {
    				if (c.size() != 5) {
    					try {
    						lock.wait();
    					} catch (InterruptedException e) {
    						e.printStackTrace();
    					}
    				}
    				System.out.println("size is 5");
    				// 通知t1醒过来，继续打印
    				lock.notify();
    			}
    		}, "t2").start();
    		new Thread(() -> {
    			synchronized (lock) {
    				for (int i = 0; i < 10; i++) {
    					c.add(new Object());
    					System.out.println("add " + i);
    					if (c.size() == 5) {
    						lock.notify();
    						try {
    							// 释放锁，让t2先执行
    							lock.wait();
    						} catch (InterruptedException e) {
    							e.printStackTrace();
    						}
    					}
    					try {
    						TimeUnit.SECONDS.sleep(1);
    					} catch (InterruptedException e) {
    						e.printStackTrace();
    					}
    				}
    			}
    		}, "t1").start();
    	}
    }

由于`notify`并不会释放锁，所以在 **t1** `notify`后紧接着`wait`，并在 **t2** 结束时通知 **t1**，这样显得很繁琐，更简洁的一种方法是使用`CountDownLatch`。

    public class MyContainer3 {
    	List<Object> list = new ArrayList<>();

    	public void add(Object o) {
    		list.add(o);
    	}

    	public int size() {
    		return list.size();
    	}

    	public static void main(String[] args) {
    		CountDownLatch count = new CountDownLatch(1);
    		MyContainer3 c = new MyContainer3();
    		new Thread(() -> {
    			if (c.size() != 5) {
    				try {
    					count.await();
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    			System.out.println("size is 5");
    		}, "t2").start();
    		new Thread(() -> {
    			for (int i = 0; i < 10; i++) {
    				c.add(new Object());
    				System.out.println("add " + i);
    				if (c.size() == 5) {
    					count.countDown();
    				}
    				try {
    					TimeUnit.SECONDS.sleep(1);
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    		}, "t1").start();
    	}
    }

`count`调用`countDown()`后值为0，阻塞线程 **t2** 被唤醒。

### 参考资料
> https://www.youtube.com/watch?v=_D7MiAEM3oY  
《深入理解Java虚拟机》
