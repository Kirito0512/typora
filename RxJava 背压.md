### 1. 背压的问题背景(起因)

- 上下游在不同的线程中
- 上游发送数据的速度快于下游接受处理数据的速度
- 没来得及处理的数据，存放在异步缓存池中
- 缓存池越积越多，最后会内存溢出



### 2. RxJava2的背压解决方案

#### 2.1 Flowable

- Flowable 是 Publisher 与 Subscriber 这组观察者模式中的 Publisher 的典型实现

- Flowable是在Observable基础上优化背压后的产物，Observable能解决的问题，Flowable也都能解决。
- 由于Flowable发射的数据流，以及数据加工处理的操作符，要兼顾背压支持，而附加了额外的逻辑。Flowable的运行效率要明显低于Observable

- **只有在处理背压问题时，才需要使用Flowable**



#### 2.2 BackpressureStrategy参数

> Flowable.create时传入，系统提供了5个选择

- MISSING——

  >  MissingEmitter，相当于不指定背压策略，不会对通过onNext发射的数据做缓存或丢弃处理。需要下游通过背压操作符(onBackpressureBuffer/onBackpressureDrop/onBackpressureLatest)指定背压策略

- ERROR

- BUFFER——BufferAsyncEmitter,Flowable处理背压的默认策略

- DROP

- LATEST----LatestAsyncEmitter



#### 2.3 Subscription

- 订阅器的onSubscribe方法，参数不是Observable中对应的Disposable，而是Subscription。并且多了一个request方法。

- Subscription与Disposable都是，观察者与可观察对象建立订阅状态后，回调回来的参数。

- Disposable使用dispose()方法，Subscription使用cancel()方法，都可以取消相应的订阅关系。
- Subscription还有一个request方法，用于表示下游对数据的请求数量(响应式拉取)。如果不显式调用，那么上游会默认下游需求量为0，Flowable发送的数据不会交给下游处理。





#### 2.4 Flowable的异步缓存池

> 当上游发送数据的速度快于下游接收数据的速度，且二者运行在不同线程中时，Flowable会把拉布机处理的数据，缓存到自身的异步缓存池。
>
> 缓存池的容量上限，是128





### 参考

[Rxjava2入门教程五：Flowable背压支持——对Flowable最全面而详细的讲解](https://www.jianshu.com/p/ff8167c1d191)

