---
layout:     post
title:      "Java并发知识点总结（四）"
subtitle:   "ThreadPoolExecutor, ForkJoin"
date:       2018-08-01
author:     "Jon Lee"
header-img: "img/in-post/2018-07-27-java-concurrency-summary/bg4.jpg"
catalog: true
categories : Java并发
tags:
    - Java
    - 并发
---

### ThreadPoolExecutor

通过一个小例子来回顾一下线程池。  
创建一个固定线程数量的线程池`newFixedThreadPool`，然后打印结果。

    public class TestThreadPool {
    	public static void main(String[] args) {
    		ExecutorService service = Executors.newFixedThreadPool(5);
    		for (int i = 0; i < 8; i++) {
    			service.execute(() -> {
    				try {
    					TimeUnit.SECONDS.sleep(1);
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    				System.out.println(Thread.currentThread().getName());
    			});
    		}
    		System.out.println("service: " + service);
    		service.shutdown();
    		System.out.println("service.isTerminated() " + service.isTerminated());
    		System.out.println("service.isShutdown() " + service.isShutdown());
    		System.out.println("service: " + service);
    		try {
    			TimeUnit.SECONDS.sleep(3);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		System.out.println("service.isTerminated() " + service.isTerminated());
    		System.out.println("service.isShutdown() " + service.isShutdown());
    		System.out.println("service: " + service);
    	}
    }

*Output*
>service: java.util.concurrent.ThreadPoolExecutor@e9e54c2[Running, pool size = 5, active threads = 5, queued tasks = 3, completed tasks = 0]  
service.isTerminated() false  
service.isShutdown() true  
service: java.util.concurrent.ThreadPoolExecutor@e9e54c2[Shutting down, pool size = 5, active threads = 5, queued tasks = 3, completed tasks = 0]  
pool-1-thread-1  
pool-1-thread-2  
pool-1-thread-3  
pool-1-thread-4  
pool-1-thread-5  
pool-1-thread-1  
pool-1-thread-2  
pool-1-thread-3  
service.isTerminated() true  
service.isShutdown() true  
service: java.util.concurrent.ThreadPoolExecutor@e9e54c2[Terminated, pool size = 0, active threads = 0, queued tasks = 0, completed tasks = 8]

当我们创建线程池的时候，调用的是`Executors`的工厂方法，实际上创建的是一个`ThreadPoolExecutor`对象。  
`ThreadPoolExecutor`主要由四个组件构成：

* `corePool` 核心线程池大小
* `maximumPool` 最大线程池大小
* `BlockingQueue` 用来暂时保存任务的工作队列
* `RejectedExecutionHandler` 当`ThreadPoolExecutor`关闭或者饱和时，`execute`方法将调用`Handler`。

`Executors`提供了三种类型的`ThreadPoolExecutor`，包括`FixedThreadPool`、`SingleThreadExecutor`和`CachedThreadPoolExecutor`。

### FutureTask

`FutureTask`实现`RunnableFuture`接口，`RunnableFuture`又继承自`Runnable`和`Future`接口。  
执行`ExecutorService.submit`方法会返回实现了`Future`接口的对象，也就是`FutureTask`对象（Java7添加了`ForkJoinTask`，所以`FutureTask`并不是`Future`接口的唯一实现类）java。  
由于`FutureTask`也实现了`Runnable`接口，所以也可以自己创建，然后交给线程执行。  
主线程可以通过`get`方法得到结果，或者执行`cancel`方法取消任务的执行。

    public class TestFuture {
    	public static void main(String[] args) throws InterruptedException, ExecutionException {
    		FutureTask<Integer> task = new FutureTask<>(() -> {
    			TimeUnit.SECONDS.sleep(1);
    			return 1;
    		});
    		new Thread(task).start();
    		System.out.println(task.get()); // 阻塞

    		ExecutorService service = Executors.newFixedThreadPool(5);
    		Future<Integer> future = service.submit(() -> {
    			TimeUnit.SECONDS.sleep(1);
    			return 1;
    		});
    		// future.cancel(true);
    		System.out.println(future.isDone());
    		System.out.println(future.get());
    		System.out.println(future.isDone());

    		service.shutdown();
    	}
    }

*Output*
>1  
false  
1  
true

### 并行计算

如何利用 **线程池** 和 **Future** 实现并行计算。  
**问题是我们要计算1~1,000,000之间的质数。**  
已知一共有 **78,498** 个质数，下面来看看如何实现，然后校验一下结果是否正确。

    public class ParallelComputing {
    	private static final int CPU_CORE_NUM = 4;
    	private static final int N = 1000000;

    	public static void main(String[] args) {
    		Instant start = Instant.now();
    		System.out.println(getPrimes(1, N).size());
    		Instant end = Instant.now();
    		System.out.println("time cost" + Duration.between(start, end).toMillis());
    	}

    	static boolean isPrime(int n) {
    		if (n < 2) {
    			return false;
    		}
    		if (n == 2) {
    			return true;
    		}
    		if (n % 2 == 0) {
    			return false;
    		}
    		for (int i = 3; i * i <= n; i += 2) {
    			if (n % i == 0) {
    				return false;
    			}
    		}
    		return true;
    	}

    	static List<Integer> getPrimes(int start, int end) {
    		List<Integer> res = new ArrayList<>();
    		for (int i = start; i <= end; i++) {
    			if (isPrime(i)) {
    				res.add(i);
    			}
    		}
    		return res;
    	}
    }

*Output*
>78498  
time cost 159

上面的程序是用一个线程计算，结果正确，耗时159ms，速度还是比较快的。  
下面用四个线程来计算，看看是否会快一些。

    public class ParallelComputing {
    	private static final int CPU_CORE_NUM = 4;

    	public static void main(String[] args) throws InterruptedException, ExecutionException {

    		ExecutorService service = Executors.newFixedThreadPool(CPU_CORE_NUM);
    		Future<List<Integer>> f1 = service.submit(new MyTask(1, 250000));
    		Future<List<Integer>> f2 = service.submit(new MyTask(250001, 500000));
    		Future<List<Integer>> f3 = service.submit(new MyTask(500001, 750000));
    		Future<List<Integer>> f4 = service.submit(new MyTask(750001, 1000000));

    		Instant start = Instant.now();
    		System.out.println(f1.get().size() + f2.get().size() + f3.get().size() + f4.get().size());
    		Instant end = Instant.now();
    		System.out.println("time cost " + Duration.between(start, end).toMillis());

    		// service.submit(new MyTask(1, 80000));

    		service.shutdown();
    	}

    	static class MyTask implements Callable<List<Integer>> {
    		int start, end;

    		public MyTask(int start, int end) {
    			this.start = start;
    			this.end = end;
    		}

    		@Override
    		public List<Integer> call() throws Exception {
    			return getPrimes(start, end);
    		}
    	}
        // isPrime()
        // getPrimes()
    }

*OutPut*
>78498  
time cost 62

可以看到多线程并行计算下确实提高了运行速度，耗时减少到了62ms。

再用一下`埃式筛法`，看看如果只用单线程计算最快需要多久。

    class primesCounter {
    	private static final int N = 1000000;

    	public static void main(String[] args) {
    		Instant start = Instant.now();
    		// 埃式筛法
    		System.out.println(countPrimesWithEratosthenes(N).size());
    		Instant end = Instant.now();
    		long d = Duration.between(start, end).toMillis();
    		System.out.println("time cost: " + d);
    	}

    	private static List<Integer> countPrimesWithEratosthenes(int n) {
    		List<Integer> res = new ArrayList<>();
    		if (n < 2) {
    			return res;
    		}
    		boolean[] isPrime = new boolean[n + 1];
    		for (int i = 2; i <= n; i++) {
    			isPrime[i] = true;
    		}
    		for (int i = 2; i * i <= n; i++) {
    			if (!isPrime[i]) {
    				continue;
    			} else {
    				for (int j = i * i; j <= n; j += i) {
    					isPrime[j] = false;
    				}
    			}
    		}
    		for (int i = 2; i <= n; i++) {
    			if (isPrime[i]) {
    				res.add(i);
    			}
    		}
    		return res;
    	}
    }

*Output*
>78498  
time cost: 21

三种方法，`埃式筛法`是最快的，因为时间复杂度要比正常去计算一个数是不是素数，再往里添加的方法小了很多。也可以看出，相同方法下多线程的并行计算要比单线程快了很多。

Java8提供了`parallelStream`，并行对流中的元素进行计算，它是以`ForkJoin`框架为基础的，下面来看一下什么是`ForkJoin`。

### ForkJoin

Java7提出的`ForkJoin`是一个用于并行执行任务的框架，把一个大任务分成若干的消任务，最终汇总每个小任务的结果得到大任务的结果。  

下面的程序就是利用`ForkJoinPool`实现一个求和任务。首先定义一个`ForkJoinTask`类，继承`RecursiveTask`或者`RecursiveAction`，两个类分别代表有返回值和没有返回值。然后定义每次计算的 **阈值** （阈值的设定会影响性能），超过阈值就将任务分割`fork`，当所有任务计算完后，再`join`回来，和递归的思想是一样的。

由于各种因素，即便任务拆分是平均的，也不能保证所有子任务能同时执行结束， 大部分情况是某些子任务已经结束， 其他子任务还有很多， 在这个时候就会有很多资源空闲， 所以`ForkJoin`框架通过工作窃取机制来保证资源利用最大化， 让空闲的线程去偷取正在忙碌的线程的任务。

    public class TestForkJoin {

    	public static void main(String[] args) throws InterruptedException, ExecutionException {
    		ForkJoinPool fjp = new ForkJoinPool(4); // 最大并发数4
    		long startTime = System.currentTimeMillis();
    		ForkJoinTask<Long> task = new SumTask(1, 100000);
    		fjp.execute(task);
    		Long result = task.join();
    		long endTime = System.currentTimeMillis();
    		System.out.println("Fork/join sum: " + result + " in " + (endTime - startTime) + " ms.");
    	}
    }

    class SumTask extends RecursiveTask<Long> {

    	static final int THRESHOLD = 10000;
    	int start;
    	int end;

    	SumTask(int start, int end) {
    		this.start = start;
    		this.end = end;
    	}

    	@Override
    	protected Long compute() {
    		if (end - start <= THRESHOLD) {
    			// 如果任务足够小,直接计算:
    			long sum = 0;
    			for (int i = start; i <= end; i++) {
    				sum += i;
    			}
    			System.out.println(String.format("compute %d~%d = %d", start, end, sum));
    			return sum;
    		}
    		// 任务太大,一分为二:
    		int middle = (end + start) / 2;
    		System.out.println(String.format("split %d~%d ==> %d~%d, %d~%d", start, end, start, middle, middle, end));
    		SumTask subtask1 = new SumTask(start, middle);
    		SumTask subtask2 = new SumTask(middle + 1, end);
    		// 执行子任务
    		subtask1.fork();
    		subtask2.fork();
    		// 等待子任务执行完
    		Long subresult1 = subtask1.join();
    		Long subresult2 = subtask2.join();
    		// 合并子任务
    		Long result = subresult1 + subresult2;
    		System.out.println("result = " + subresult1 + " + " + subresult2 + " ==> " + result);
    		return result;
    	}
    }

### 参考资料

> 《Java并发编程的艺术》  
https://www.youtube.com/watch?v=_D7MiAEM3oY&list=PL0onFhfJfEDD2KcgyXlAFg20XDLabnXqz

---
