### 1. JMM java内存模型

<img src="https://tva1.sinaimg.cn/large/00831rSTly1gd2tdjlnkrj30m00in3zq.jpg" style="zoom:50%;" />

在JVM内存结构的图中，**Java堆** 和 **方法区** 的是多线程共享的数据区域。

也就是说，多个线程可以操作，保存在堆或者方法区中的同一个数据。

这也就是我们常说的“Java的线程间通过共享内存进行通信”。



而JMM并不像JVM内存结构那样真实存在，他只是一个抽象概念。主要是应对CPU速度远大于内存读取速度。



在 JMM 中，我们把多个线程间通信的共享内存称为 **主内存**。

在并发编程中，多个线程都维护了一个自己的 **本地内存** (这是个抽象概念)，其中保存的数据是主内存中的 **数据拷贝**。

JMM 主要是控制本地内存和主内存之间的数据交互的。



### 2. Synchronized



![img](https://tva1.sinaimg.cn/large/00831rSTly1gcz3vmz6irj30rl0gajtj.jpg)

> Synchronized 底层是通过，对象内部的监视器锁（monitor）来实现的。监视器锁又依赖于，底层操作系统的MutexLock来实现。
>
> 操作系统实现线程之间的切换，需要从用户态转换到核心态，成本非常高。这也是synchronized效率低的原因。

#### 2.1 synchronized修饰代码块

从反编译来看，对代码块的修饰，是通过 `monitorenter` 和 `monitorexit` 指令完成的

简单来说，每个对象都有一个监视器锁（monitor）

进入代码块时，系统会获取monitor的所有权，如果获取成功，就能执行对应逻辑，执行完毕后放弃所有权。别的线程可以继续访问代码块。如果已经有别的线程持有所有权，当前线程会进入阻塞状态。

> 如果一个线程已经占有一个对象的monitor，再次尝试 `monitorenter`，那么会**重新进入**，把monitor的进入数 +1
>
> 对应的 `monitorexit` 会把monitor进入数-1，如果此时进入数为0，那么别的线程可以尝试获取monitor的所有权

具体可以看 [Java并发编程：Synchronized及其实现原理](https://www.cnblogs.com/paddix/p/5367116.html)

#### 2.2 synchronized修饰方法

synchronized方法，相对于普通方法，在反编译结果的常量池里，多了 ACC_SYNCHRONIZED 标识符。

JVM就是根据该标示符来实现方法的同步的：

当方法调用时，会检查 ACC_SYNCHRONIZED 访问标志是否被设置。如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体。

方法执行完后再释放monitor。

#### 2.4 synchronized的作用

保障原子性、可见性和有序性

- 锁是通过互斥保障原子性的
- 锁通过写线程 flush（冲刷）处理器缓存 & 读线程 refresh（刷新）处理器缓存 来实现可见性。Java平台中，锁的获得隐含刷新处理器缓存的动作，使得 `read thread` 在执行临界区代码前（获得锁之后），可以将 `write thread` 对共享变量所做的更新同步到该线程的高速缓存中。同理，锁的释放隐含 flush处理器缓存的动作，使得写线程对共享变量的更新能够被 “推送” 到处理器的高速缓存中。
- 锁能够保障有序性。写线程在临界区的一系列操作，在读线程执行的临界区看起来就像是完全按照源代码顺序执行的。即 “读线程对这些操作的感知顺序与源代码顺序一致”。同时，在临界区内部，其实代码还是可以重排序的。

#### 2.3 synchronized的优劣势

- 内部锁

  synchronized属于内部锁，内部锁是一种排它锁，它能够保障原子性、可见性和有序性（但只能保障临界区的代码不和临界区外的代码重排序，临界区内的代码还是可以重排序的，所以才有DCL里的问题）

- 显式锁

  显式锁的作用与内部锁相同，但是提供了一些内部锁不具备的特性，比如**ReentrantLock**

1. 内部锁  简单易用，且不会导致锁泄露

2. 内部锁仅支持非公平锁，显式锁即支持非公平锁也支持公平锁，而公平锁开销更大，所以显式锁默认为非公平锁。

3. 内部锁的申请与释放只能是在一个方法内进行（因为代码块无法跨方法），而显式锁支持在一个方法内申请锁，却在另一个方法里释放锁。

4. 如果一个内部锁的持有线程一直不释放锁，那么同步在该锁上的所有线程会一直被block，而使其任务无法进展。而ReenrantLock的tryLock在锁被其他线程持有时会return false。

### 3. volatile

#### 3.1 volatile作用

volatile 是轻量级锁，表示被修饰的变量的值容易变化（即被其他线程更改）

它的作用包括防止重排序 + 实现可见性 + 保证（long / double）单次读/写本身的原子性

- volatile变量不会被编译器分配到寄存器进行存储，对volatile变量的读写操作都是内存访问（访问高速缓存相当于主内存）





#### 3.2 有序性实现

了解一下Java中的happen-before规则

在6种符合happens-before的情形下，JMM可以保证跨进程的内存可见性。

值得注意的是，其中一条就和volatile相关

- 同一个线程中的，前面的操作 happen-before 后续的操作
- 监视器上的解锁操作 happen-before 其后续的加锁操作
- 对volatile变量的写操作 happen-before 后续的读操作 （volatile规则）
- 线程的start方法 happen-before 该线程所有后续操作
- 线程所有操作 happen-before 其他线程在该线程上调用join 返回成功后的操作
- 如果 a happen-before b，b happen-before c，则a happen-before c

#### 3.3 实现原理

volatile 对于可见性和有序性，是通过两类 内存屏障 Barrier 实现的。

可见性 （将新的值更新到线程的工作内存）-> loadBarrier & storeBarrier

有序性（防止重排序）-> acquireBarrier & releaseBarrier

- loadBarrier -> 读操作 -> acquireBarrier
- releaseBarrier -> 写操作 -> storeBarrier

[volatile如何保证可见性与有序性](https://since1986.github.io/5ab80f4c.html)

