---
layout:     post
title:      "Java并发知识点总结（三）"
subtitle:   "ConcurrentMap, BlockingQueue"
date:       2018-07-31
author:     "Jon Lee"
header-img: "img/in-post/2018-07-27-java-concurrency-summary/bg3.jpg"
catalog: true
categories : Java并发
tags:
    - Java
    - 并发
---

### 火车站卖票问题

模拟火车站卖票，由于多个用户从若干个窗口买票肯定会出现并发，所以要使用同步。

使用`synchronized`加锁，每次只能有一个线程拿到锁，效率很低。

    public class TicketSeller1 {
    	static List<String> tickets = new ArrayList<>();

    	static {
    		for (int i = 0; i < 1000; i++) {
    			tickets.add("No. " + i);
    		}
    	}

    	public static void main(String[] args) {
    		for (int i = 0; i < 10; i++) {
    			new Thread(() -> {
    				while (true) {
    					synchronized (tickets) {
    						if (tickets.size() <= 0) {
    							break;
    						}
    						try {
    							TimeUnit.MILLISECONDS.sleep(10);
    						} catch (InterruptedException e) {
    							e.printStackTrace();
    						}
    						System.out.println(Thread.currentThread().getName() + " 销售了 " + tickets.remove(0));
    					}
    				}
    			}, "t" + i).start();
    		}
    	}
    }

使用并发容器，效率更高。

    public class TicketSeller2 {

    	static Queue<String> tickets = new ConcurrentLinkedQueue<>();

            // ...

    	public static void main(String[] args) {
    		for (int i = 0; i < 10; i++) {
    			new Thread(() -> {
    				while (true) {
    					String s = tickets.poll();
    					if (s == null) {
    						break;
    					} else {
    						System.out.println(Thread.currentThread().getName() + " 销售了 " + s);
    					}
    				}
    			}, "t" + i).start();
    		}
    	}
    }

早期Java是通过实现 **同步容器**，来提高线程安全性，例如`Hashtable`和`Vector`，实现线程安全的方式就是将状态封装起来，并对每个共有方法都进行同步，使得每次只有一个线程能访问容器的状态。  
**Java 5.0**  提供了多种并发容器来改进同步容器的性能，同步容器将所有对容器状态的访问都串行化，这种方法严重降低了并发性，当有多个线程竞争容器的锁时，吞吐量将严重降低。  

**通过并发容器来代替同步容器，可以极大地提高伸缩性并降低风险。**

下面来挨个总结一下并发容器。  

### ConcurrentMap

`ConcurrentMap`接口有两个实现类，`ConcurrentHashMap`和`ConcurrentSkipListMap`。
Java中还有一个存储键值对的容器就是上面提到的`Hashtable`。
`ConcurrentSkipListMap`使用跳表实现，是一个高并发下使用的排序Map。
如果不排序的话，使用`Hashtable`或者`ConcurrentHashMap`，那么这两种容器效率如何，下面来做个小实验。

    public class ConcurrentMap {

    	public static void main(String[] args) {

    		Map<Integer, Integer> map = new ConcurrentHashMap<>();
    		// Map<Integer, Integer> map = new Hashtable<>();

    		Random rand = new Random();
    		Thread[] ths = new Thread[100];
    		CountDownLatch latch = new CountDownLatch(ths.length);
    		Instant start = Instant.now();
    		for (int i = 0; i < ths.length; i++) {
    			ths[i] = new Thread(() -> {
    				for (int j = 0; j < 10000; j++) {
    					map.put(rand.nextInt(100000), rand.nextInt(100000));
    				}
    				latch.countDown();
    			});
    		}
    		Arrays.asList(ths).forEach((t) -> t.start());
    		try {
    			latch.await();
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		Instant end = Instant.now();
    		Duration d = Duration.between(start, end);
    		System.out.println(d.toMillis());
    	}
    }

*Output*
> ConcurrentHashMap: 369ms  
Hashtable: 553ms

多运行几次会发现虽然每次的结果不同，但是`ConcurrentHashMap`比`Hashtable`要快。  
原因是`Hashtable`使用的是`synchronzied`实现，而`ConcurrentHashMap`使用了分段锁。

### CopyOnWriteList

`CopyOnWriteArrayList`用于替代同步`List`,某些情况下并发性能更好。  
**写入时复制** 容器在每次修改时，都会创建并重新发布一个新的容器副本，再将引用指向新的容器（原子操作）。这样做的好处是我们可以对`CopyOnWrite`容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。

    public class CopyOnWriteList {
    	public static void main(String[] args) {

    		List<Integer> list = Collections.synchronizedList(new ArrayList<>());
    		// List<Integer> list = new Vector<>();
    		// List<Integer> list = new CopyOnWriteArrayList<>();

    		Thread[] ths = new Thread[100];

    		// 写操作测试
    		CountDownLatch latch1 = new CountDownLatch(ths.length);
    		for (int i = 0; i < ths.length; i++) {
    			ths[i] = new Thread(() -> {
    				for (int j = 0; j < 10000; j++) {
    					list.add(j);
    				}
    				latch1.countDown();
    			});
    		}
    		Instant start = Instant.now();
    		Arrays.asList(ths).forEach((t) -> t.start());
    		try {
    			latch1.await();
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		Instant end = Instant.now();
    		Duration d = Duration.between(start, end);
    		System.out.println("write time " + d.toMillis());
    		System.out.println("size " + list.size());

    		// 读操作性能测试
    		CountDownLatch latch2 = new CountDownLatch(ths.length);
    		for (int i = 0; i < ths.length; i++) {
    			ths[i] = new Thread(() -> {
    				for (int j = 0; j < list.size(); j++) {
    					list.get(j);
    				}
    				latch2.countDown();
    			});
    		}
    		start = Instant.now();
    		Arrays.asList(ths).forEach((t) -> t.start());
    		try {
    			latch2.await();
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		end = Instant.now();
    		d = Duration.between(start, end);
    		System.out.println("read time " + d.toMillis());
    	}
    }

*Output*

容器类型|写操作时间（ms）|读操作时间（ms）
-------|---------------|--------------
Collections.synchronizedList(ArrayList) |64|1316
Vector|56|964
CopyOnWriteArrayList|7806|719

可以看到，`CopyOnWriteArrayList`在进行写操作时非常地慢，但是读取数据时很快，所以`CopyOnWrite`并发容器用于 **读多写少** 的并发场景。

### ConcurrentQueue

`ConcurrentLinkedQueue`是一个基于链接节点的无界线程安全队列，使用 **非阻塞** 的方式实现线程安全，通过 **CAS** 来实现。具体看这篇播客吧[ConcurrentLinkedQueue的实现原理分析](http://www.infoq.com/cn/articles/ConcurrentLinkedQueue)，目前还没太弄明白。

### BlockingQueue

阻塞队列顾名思义就是支持阻塞的插入和移除方法的队列。  
**阻塞队列** 常用于生产者和消费者的场景。  
插入和移除的四种操作方式：

方法/处理方式|抛出异常|返回特殊值|一直阻塞|超时退出
------------|-------|---------|--------|-------
插入方法     |add(e) |offer(e) |put(e)  |offer(e, time, unit)
移除方法     |remove()|poll()  |take()  |poll(e, time, unit)
处理方式     |element()|peek() |-       |-

下面用`LinkedBlockingQueue`来实现一个生产者消费者程序。

    public class TestLinkedBlockingQueue {

    	private static final BlockingQueue<String> queue = new LinkedBlockingQueue<>();

    	public static void main(String[] args) {
    		// 创建生产者线程
    		for (int i = 0; i < 2; i++) {
    			new Thread(() -> {
    				for (int j = 0; j < 100; j++) {
    					try {
    						queue.put(Thread.currentThread().getName() + " " + j);
    					} catch (InterruptedException e) {
    						e.printStackTrace();
    					}
    				}
    			}, "p" + i).start();
    		}
    		// 创建消费者线程
    		for (int i = 0; i < 5; i++) {
    			new Thread(() -> {
    				for (;;) {
    					try {
    						System.out.println(Thread.currentThread().getName() + " " + queue.take());
    					} catch (InterruptedException e) {
    						// TODO Auto-generated catch block
    						e.printStackTrace();
    					}
    				}
    			}, "c" + i).start();
    		}
    	}
    }

`LinkedBlockingQueue`的默认和最大长度是`Integer.MAX_VALUE`。  
也可以使用`ArrayBlockingQueue`创建一个由数组构成的有界阻塞队列。  

    ArrayBlockingQueue fairQueue = new ArrayBlockingQueue(1000, true);

`fairQueue`设置容量为1000，并且锁设置为公平锁，使每个线程按照阻塞顺序公平地访问队列。

`PriorityBlockingQueue`是线程安全的优先队列实现。

`DelayQueue`是一个延时获取元素的无界阻塞队列。队列中的元素必须实现`Delayed`接口，在创建元素时可以指定多久才能从队列中获取元素。  
`getDelay`方法返回的是当前元素还需要延时多长时间，`compareTo`方法指定元素在队列中的顺序。

    public class TestDelayQueue {

    	private static BlockingQueue<MyTask> tasks = new DelayQueue<>();

    	private static class MyTask implements Delayed {
    		Long runningTime;

    		MyTask(long rt) {
    			this.runningTime = rt;
    		}

    		@Override
    		public int compareTo(Delayed o) {
    			if (this.getDelay(TimeUnit.MILLISECONDS) < o.getDelay(TimeUnit.MILLISECONDS)) {
    				return -1;
    			} else if (this.getDelay(TimeUnit.MILLISECONDS) > o.getDelay(TimeUnit.MILLISECONDS)) {
    				return 1;
    			} else {
    				return 0;
    			}
    		}

    		@Override
    		public long getDelay(TimeUnit unit) {
    			return unit.convert(runningTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    		}

    		@Override
    		public String toString() {
    			return "" + runningTime;
    		}
    	}

    	public static void main(String[] args) {
    		long now = System.currentTimeMillis();
    		MyTask t1 = new MyTask(now);
    		MyTask t2 = new MyTask(now + 1000);
    		MyTask t3 = new MyTask(now + 500);
    		MyTask t4 = new MyTask(now + 2000);
    		MyTask t5 = new MyTask(now + 1500);

    		try {
    			tasks.put(t1);
    			tasks.put(t2);
    			tasks.put(t3);
    			tasks.put(t4);
    			tasks.put(t5);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}

    		for (int i = 0; i < 5; i++) {
    			try {
    				System.out.println(Long.parseLong(tasks.take().toString()) - now);
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			}
    		}
    	}
    }

*Output*
>0  
500  
1000  
1500  
2000

可以看到延时最长的任务最后出队，所以`DelayQueue`可以在以下场景应用：

* 缓存系统的设计
* 定时任务的调度

`SynchronousQueue`是一个容量为0的阻塞队列，每一个`put`操作必须等待一个`take`操作，否则不能继续添加元素。

`LinkedTransferQueue`比其它阻塞队列多了`transfer`和`tryTransfer`两个方法，`transfer`方法是生产者把传入的元素立刻传输给消费者，如果当前有消费者等待接收元素则直接传递，如果没有，生产者线程会把元素放在队列的tail节点，直到有消费者消费才返回。  
`tryTransfer`方法作用是看看当前生产者传入的元素是否能够直接传输给消费者，能的话返回`true`，没有消费者等待消费则返回`false`。

### 参考资料

> 《Java并发编程实战》   
《Java并发编程的艺术》  
https://www.youtube.com/watch?v=_D7MiAEM3oY&list=PL0onFhfJfEDD2KcgyXlAFg20XDLabnXqz

---
