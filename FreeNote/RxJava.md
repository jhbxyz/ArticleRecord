# RxJava 基础且重要的知识点

### 1.RxJava 基础

异步：线程切换

简洁： **随着程序逻辑变得越来越复杂，它依然能够保持简洁**

RxJava 的异步实现，是通过一种**扩展的观察者模式**来实现的

##### 1.基本概念

- `Observable` (可观察者，即被观察者)

- `Observer` (观察者)

- `subscribe` (订阅)、事件

  `Observable` 和 `Observer` 通过 `subscribe()` 方法实现订阅关系，从而 `Observable` 可以在需要的时候发出事件来通知

##### 2.事件

- `onCompleted()`: 事件队列完结。RxJava 不仅把每个事件单独处理，还会把它们看做一个队列。RxJava 规定，当不会再有新的 `onNext()` 发出时，需要触发 `onCompleted()` 方法作为标志。
- `onError()`: 事件队列异常。在事件处理过程中出异常时，`onError()` 会被触发，同时队列自动终止，不允许再有事件发出。
- 在一个正确运行的事件序列中, `onCompleted()` 和 `onError()` 有且只有一个，并且是事件序列中的**最后一个**。需要注意的是，`onCompleted()` 和 `onError()` 二者也是互斥的，即在队列中调用了其中一个，就不应该再调用另一个

##### 3.Scheduler 线程控制

- `Schedulers.immediate()`: 直接在当前线程运行，相当于不指定线程。这是默认的 `Scheduler`。
- `Schedulers.newThread()`: 总是启用新线程，并在新线程执行操作。
- `Schedulers.io()`: I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 `Scheduler`。行为模式和 `newThread()` 差不多，区别在于 `io()` 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 `io()` 比 `newThread()` 更有效率。不要把计算工作放在 `io()` 中，可以避免创建不必要的线程。
- `Schedulers.computation()`: 计算所使用的 `Scheduler`。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 `Scheduler` 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 `computation()` 中，否则 I/O 操作的等待时间会浪费 CPU。
- Android 还有一个专用的 `AndroidSchedulers.mainThread()`，它指定的操作将在 Android 主线程运行。
- `subscribeOn()`  事件产生的线程
- `observeOn()`  `Subscriber` 所运行在的线程。或者叫做事件消费的线程

### 2.RxJava 的观察者模式

##### 2.1两种观者着模式

- Observable(被观察者)/Observer（观察者）—>不支持背压
- Flowable(被观察者)/Subscriber(观察者)   —>**支持背压**

##### 2.2.其他观察者

- Single/SingleObserver
- Completable/CompletableObserver
- Maybe/MaybeObserver 发送单个数据



##### 2.3背压

背压是指在**异步**场景中，被观察者发送事件速度**远快于**观察者的处理速度的情况下，一种告诉上游的被观察者降低发送速度的**策略**

Observable 不支持背压，啥叫不支持背压呢？

被观察者快速发送大量数据时，下游不会做其他处理，即使数据大量堆积，调用链也**不会**报MissingBackpressureException,**消耗内存过大只会OOM**

使用 Observable 考虑数据量是不是很大(官方给出以**1000**个事件为分界线，仅供参考)



