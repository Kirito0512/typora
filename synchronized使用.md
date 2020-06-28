### synchronized的几种用法

> Java语言的关键字，当它用来修饰一个方法或者一个代码块的时候，能够保证在同一时刻最多只有一个线程执行该段代码。但是，它并不能保证，可见性和有序性。

#### 1. synchronized代码块

- synchronized(this)
- synchronized(*.class)
- synchronized(任意对象)

#### 2. synchronized方法

- synchronized普通方法
- synchronized静态方法

### 3. 相关测试代码

> 上述的5中方式，都能套用下面的代码模板进行测试。
>
> 当前代码测试的是，当一个线程，访问ObjectService的一个synchronized(this) 同步代码块时，其他线程访问同一个ObjectService 中的 synchronized(*.class)代码块，是否会阻塞？
>
> 答案是 不会阻塞

ObjectService

```
public class ObjectService {

    public void methodA() {
        synchronized (this) {
            try {
                System.out.println("methodA begin thread name = " + Thread.currentThread().getName() + " times = " + System.currentTimeMillis());
                Thread.sleep(4000);
                System.out.println("methodA end thread name = " + Thread.currentThread().getName() + " times = " + System.currentTimeMillis());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public void methodB() {
        synchronized (ObjectService.class) {
            System.out.println("methodB begin thread name = " + Thread.currentThread().getName() + " times = " + System.currentTimeMillis());
            System.out.println("methodB end thread name = " + Thread.currentThread().getName() + " times = " + System.currentTimeMillis());
        }
    }

}

```

ThreadA

```
public class ThreadA extends Thread {
    private ObjectService objectService;

    public ThreadA(ObjectService objectService) {
        super();
        this.objectService = objectService;
    }

    @Override
    public void run() {
        super.run();
        objectService.methodA();
    }
}
```

ThreadB

```
public class ThreadB extends Thread {
    private ObjectService objectService;

    public ThreadB(ObjectService objectService) {
        this.objectService = objectService;
    }

    @Override
    public void run() {
        super.run();
        objectService.methodB();
    }
}
```



Main

```
public class Main {
    public static void main(String[] args) {
        ObjectService objectService = new ObjectService();
        ThreadA threadA = new ThreadA(objectService);
        ThreadB threadB = new ThreadB(objectService);

        threadA.start();
        threadB.start();
    }
}
```

### 4. 测试结论

#### 4.1 

> 多个线程，调用同一个对象中的，不同名称的synchronized同步方法，或synchronized(this) 同步代码块时，是同步的（互相会阻塞，必须一个一个来）

#### 4.2

> 使用synchronized(任意对象) 方式，对象监视器必须为同一个对象，不然也是异步执行。这种方式与synchronized(this) ，synchronized方法之间是异步执行

#### 4.3

> synchronized 静态方法，是对 当前class持锁，与 synchronized(当前.class) 之间是同步的

#### 4.4

> synchronized(*.class) 持有的class锁，会对所有的对象实例起作用。也就是说，即使main方法中，创建两个ObjectService的对象，分别传入两个thread执行对应方法，也会是同步的。



### 参考

> 估计自己过段时间肯定会看不懂的，那就看下面的链接吧！

[synchronized(this)、synchronized(class)与synchronized(Object)的区别](https://blog.csdn.net/luckey_zh/article/details/53815694)

