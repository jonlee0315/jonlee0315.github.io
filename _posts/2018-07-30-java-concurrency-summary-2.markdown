---
layout:     post
title:      "Java并发知识点总结（二）"
subtitle:   "Java Concurrency Summary"
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
---
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

实现一个固定容量的同步容器，拥有`put`和`get`方法，以及`getCount`方法，能够支持2个生产者线程及10个消费者线程的阻塞调用。
