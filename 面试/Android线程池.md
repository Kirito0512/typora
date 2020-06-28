### 1. ThreadPoolExecutor

> 看一下构造函数

```java

public ThreadPoolExecutor(int corePoolSize,

int maximumPoolSize,

long keepAliveTime,

TimeUnit unit,

BlockingQueue<Runnable> workQueue,

ThreadFactory threadFactory,

RejectedExecutionHandler handler)

```

- corePoolSize 核心线程的数量

- maximumPoolSize 最大线程数量

- keepAliveTime

- 非核心线程的超时时长，非核心线程闲置事件超过keepAliveTime之后，会被回收。如果ThreadPoolExecutor的allowCoreThreadTimeOut设置为true，则核心线程也有对应超时时长

- unit keepAliveTime的时间单位

- workQueue 线程池中的任务队列，主要存储已经被提交但是尚未执行的任务。任务是由ThreadPoolExecutor的execute方法提交来的

- threadFactory 创建新线程使用的工厂，一般使用默认值即可

- handler 线程池无法执行新任务时（线程池已满或者无法成功执行任务），handler抛出rejectedExecution

### 2. workQueue

> workQueue是一个BlockingQueue类型，那么这个BlockingQueue又是什么呢？

>

>

>

> 它是一个特殊的队列，当我们从BlockingQueue中取数据时，如果BlockingQueue是空的，则取数据的操作会进入到阻塞状态，当BlockingQueue中有了新数据时，这个取数据的操作又会被重新唤醒。

>

> 同理，如果BlockingQueue中的数据已经满了，往BlockingQueue中存数据的操作又会进入阻塞状态，直到BlockingQueue中又有新的空间，存数据的操作又会被冲洗唤醒

> ————————————————

> 原文链接：https://blog.csdn.net/u012702547/article/details/52259529

#### 2.1 BlockingQueue的实现类

- ArrayBlockingQueue

- 由数组支持的有界BlockingQueue。它的构造方法接受一个int数据，表示容量。存储在其中的元素，按照FIFO先进先出的方式来存取。

- LinkedBlockingQueue

- 由链表支持的，可选界限的BlockingQueue。默认创建大小为Integer.MAX_VALUE 的 BlockingQueue，也人为可以指定大小。

- PriorityBlockingQueue

- 按照元素的Comparator来决定存取顺序，存入其中的数据必须实现Comparator接口（不允许添加空节点）

- SynchronousQueue

- 这是一个没有任何数据缓冲的，线程安全的，同步阻塞队列。在这个BlockingQueue中，每个插入操作必须等待另一个线程的对应的删除操作，反之亦然。我们可以理解为生产者和消费者互相等待，等到对方之后然后再一起离开。

### 3. ThreadPollExecutor的工作规则

1. execute一个线程之后，如果线程池中的线程数未达到核心线程数，则会立马启用一个核心线程去执行

2. execute一个线程之后，如果线程池中的线程数已经达到核心线程数，且workQueue未满，则将新线程放入workQueue中等待执行

3. execute一个线程之后，如果线程池中的线程数已经达到核心线程数但未超过非核心线程数，且workQueue已满，则开启一个非核心线程来执行任务

4. execute一个线程之后，如果线程池中的线程数已经超过规定的最大值，则拒绝执行该任务。ThreadPoolExecutor 会调用 RejectedExecutionHandler 的 rejectedExecution 方法通知调用者。

### 4. 常见的线程池的实现

Android 中常见的线程池有 FixedThreadPool, CacheThreadPool, ScheduledThreadPool 以及 SingleThreadExecutor。在 Executors.java 中都有对应的静态方法。

#### 4.1 FixedThreadPool

```java

public static ExecutorService newFixedThreadPool(int nThreads) {

return new ThreadPoolExecutor(nThreads, nThreads,

0L, TimeUnit.MILLISECONDS,

new LinkedBlockingQueue<Runnable>());

}

```

根据上面的方法可以发现，FixedThreadPool只有核心线程并且这些核心线程没有超时机制，另外任务队列也没有大小限制。能够更加快速地相应外界的请求。

#### 4.2 CachedThreadPool

```java

public static ExecutorService newCachedThreadPool() {

return new ThreadPoolExecutor(0, Integer.MAX_VALUE,

60L, TimeUnit.SECONDS,

new SynchronousQueue<Runnable>());

}

```

#### 4.3 ScheduledThreadPool

```java

public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {

return new ScheduledThreadPoolExecutor(corePoolSize);

}

```

#### 4.4 SingleThreadPool

```java

public static ExecutorService newSingleThreadExecutor() {

return new FinalizableDelegatedExecutorService

(new ThreadPoolExecutor(1, 1,

0L, TimeUnit.MILLISECONDS,

new LinkedBlockingQueue<Runnable>()));

}

```


