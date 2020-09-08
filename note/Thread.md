# 线程、线程池、线程同步

- Thread
- Runnable
- Callable
- 线程池
- synchronized
- 线程的生命周期
- volatile
- 系列化与反序列化

### 1.Thread

#### 1.1实现线程的两种方法

##### 1.1.1.继承Thread，重写 run() 方法

```kotlin
object : Thread() {
		//耗时操作
}.start()
```

##### 1.1.2.实现Runnable接口

```kotlin
Thread(Runnable {
		//耗时操作
}).start()
```

Kotlin 语法糖

```kotlin
//方法上封装了参数
thread() { 
    //耗时操作
}
```

#### 1.2.run()方法和start()方法的区别

- run()方法只是调用了Thread实例的run()方法而已，它仍然运行在**主线程**上
- start()方法会开辟一个新的线程，在新的线程上调用run()方法，此时它运行在新的线程上，由 **JVM 调用**。

### 2.Runnable

Runnable**只是一个接口**，所以单看这个**接口它和线程毫无关系**

Thread调用了Runnable接口中的方法用来在线程中执行任务

### 3.Callable

**Runnable 和 Callable** 都代表那些要在不同的线程中执行的任务

它们的主要区别是 Callable 的 call() 方法可以**返回值和抛出异常**，而 Runnable 的 run() 方法没有这些功能。Callable 可以返回装载有计算结果的 **Future 对象**

- Callable 接口下的方法是 call()，Runnable 接口的方法是 run()；
- Callable 的任务执行后可返回值，而 Runnable 的任务是不能返回值的；
- call() 方法可以抛出异常，run()方法不可以的；
- 运行 Callable 任务可以拿到一个 Future 对象，表示**异步计算的结果**。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过 Future 对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果；

<font color =#FF8C00 size =5>Thread类只支持Runnable接口 </font>

#### 3.1.FutureTask

FutureTask 实现了 Runnable 和 Future，所以兼顾两者优点，既可以在 Thread 中使用，又可以在 ExecutorService 中使用。

- FutureTask 是为了弥补 Thread 的不足而设计的
- 准确地知道线程什么时候执行完成并获得到线程执行完成后返回的结果
- FutureTask 是一种可以取消的异步的计算任务，它的计算是通过 Callable 实现的，它等价于可以携带结果的 Runnable
- 三个状态：等待、运行和完成

`在Android中充当线程的角色还有AsyncTask、HandlerThread、IntentService。它们本质上都是由Handler+Thread来构成的`

AsyncTask，它封装了线程池和Handler，主要为我们在子线程中更新UI提供便利。
HandlerThread，它是个具有消息队列的线程，可以方便我们在子线程中处理不同的事务。
IntentService，我们可以将它看做为HandlerThread的升级版，它是服务，优先级更高。

### 4.线程池

- 重用线程池中的线程，避免频繁地创建和销毁线程带来的性能消耗；
- 有效控制线程的最大并发数量，防止线程过大导致抢占资源造成系统阻塞；
- 可以对线程进行一定地管理。

#### 4.1ThreadPoolExecutor

ExecutorService是最初的线程池接口，ThreadPoolExecutor类是对线程池的具体实现

```java
public ThreadPoolExecutor(int corePoolSize,//核心线程的数量
                          int maximumPoolSize,//最大线程数
                          long keepAliveTime,//非核心线程的超时时长
                          TimeUnit unit,//枚举时间单位
                          BlockingQueue<Runnable> workQueue,//线程池中的任务队列
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

- corePoolSize 核心线程的数量，默认一直存活，避免了一般情况下CPU创建和销毁线程带来的开销，allowCoreThreadTimeOut属性设置为true，那么闲置的核心线程就会有超时策略，这个时间由keepAliveTime来设定；allowCoreThreadTimeOut默认为false，核心线程没有超时时间。
- maximumPoolSize 最大线程数，最大线程数=核心线程+非核心线程，非核心线程只有当核心线程不够用且线程池有空余时才会被创建，执行完任务后非核心线程会被销毁，数量超过最大线程数时其它任务可能就会被阻塞
- keepAliveTime 非核心线程的超时时长，当执行时间超过这个时间时，非核心线程就会被回收。当allowCoreThreadTimeOut设置为true时，此属性也作用在核心线程上。
- unit，枚举时间单位，TimeUnit。
- workQueue，线程池中的任务队列，我们提交给线程池的runnable会被存储在这个对象上。

线程池的分配遵循这样的规则：

- 当线程池中的核心线程数量未达到最大线程数时，启动一个核心线程去执行任务；
- 如果线程池中的核心线程数量达到最大线程数时，那么任务会被插入到任务队列中排队等待执行；
- 如果在上一步骤中任务队列已满但是线程池中线程数量未达到限定线程总数，那么启动一个非核心线程来处理任务；
- 如果上一步骤中线程数量达到了限定线程总量，那么线程池则拒绝执行该任务，且ThreadPoolExecutor会调用RejectedtionHandler的rejectedExecution方法来通知调用者。

#### 4.2.线程池的分类

都直接或者间接通过ThreadPoolExecutor来实现自己的功能

- FixedThreadPool
- CachedThreadPool
- ScheduledThreadPool
- SingleThreadExecutor

##### 4.2.1.FixedThreadPool

Executors的newFixedThreadPool()方法创建，它是个线程数量固定的线程池，该线程池的线程全部为核心线程，它们没有超时机制且排队任务队列无限制，因为全都是核心线程，所以响应较快，且不用担心线程会被回收。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

##### 4.2.2.CachedThreadPool

Executors的newCachedThreadPool()方法来创建，它是一个数量无限多的线程池，它所有的线程都是非核心线程，当有新任务来时如果没有空闲的线程则直接创建新的线程不会去排队而直接执行，并且超时时间都是60s，所以此线程池适合执行大量耗时小的任务。由于设置了超时时间为60s，所以当线程空闲一定时间时就会被系统回收，所以理论上该线程池不会有占用系统资源的无用线程。

```java
 public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

##### 4.2.3.ScheduledThreadPool

Executors的newScheduledThreadPool()方法来创建，ScheduledThreadPool线程池像是上两种的合体，它有数量固定的核心线程，且有数量无限多的非核心线程，但是它的非核心线程超时时间是0s，所以非核心线程一旦空闲立马就会被回收。这类线程池适合用于执行定时任务和固定周期的重复任务。

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```

##### 4.2.4.SingleThreadExecutor

Executors的newSingleThreadExecutor()方法来创建，它内部只有一个核心线程，它确保所有任务进来都要排队按顺序执行。它的意义在于，统一所有的外界任务到同一线程中，让调用者可以忽略线程同步问题。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

#### 4.3.线程池一般用法

- shutDown()，关闭线程池，需要执行完已提交的任务；
- shutDownNow()，关闭线程池，并尝试结束已提交的任务；
- allowCoreThreadTimeOut(boolen)，允许核心线程闲置超时回收；
- execute()，提交任务无返回值；
- submit()，提交任务有返回值；

### 5.Java线程同步：synchronized锁住的是代码还是对象

synchronized锁住的是括号里的对象，而不是代码。对于非static的synchronized方法，锁的就是对象本身也就是this（建议使用字节码文件对象）

我们在用synchronized关键字的时候，能缩小代码段的范围就尽量缩小，能在代码段上加同步就不要再整个方法上加同步。这叫**减小锁的粒度**，使代码更大程度的并发

**static synchronized**方法，static方法可以直接类名加方法名调用，方法中无法使用this，所以它锁的不是this，而是类的Class对象，所以，static synchronized方法也相当于全局锁，相当于锁住了代码段

<font color = "#ff0000" size = 4>答案是：synchronized锁住的是**括号里的对象**，而不是代码</font>

- 同步方法:锁的就是对象本身也就是this
- 同步代码块: 方法参数的对象 一般是 Class 对象
- 静态同步方法，同步代码块

### 7.java中的wait、notify、notifyAll

- **Object类的3个本地方法**
- **调用这3个方法的时候，当前线程必须获得这个对象的锁**
- 

wait：线程自动释放其占有的对象锁，并等待notify
notify：唤醒一个正在wait当前对象锁的线程，并让它拿到对象锁
notifyAll：唤醒所有正在wait前对象锁的线程

注意：**notify**是本地方法，具体唤醒哪一个线程由虚拟机控制；notifyAll后并不是所有的线程都能马上往下执行，它们**只是跳出了wait状态**，接下来它们还会是竞争对象锁。

### 8.CountDownLatch 控制多线程并发等待

可以让线程等待其它线程完成一组操作后才能执行，否则就一直等待

- 这个类使用一个整形参数来初始化，这个整形参数代表着等待其他线程的数量，使用await()方法让线程开始等待其他线程执行完毕

CountDownLatch并不是用来保护共享资源同步访问的，而是用来控制并发线程等待的。并且CountDownLatch只允许进入一次，一旦内部计数器等于0，再调用这个方法将不起作用，如果还有第二次并发等待，你还得创建一个新的CountDownLatch。

### 9.线程生命周期

- 新建状态:

  使用 **new** 关键字和 **Thread** 类或其子类建立一个线程对象后，该线程对象就处于新建状态。它保持这个状态直到程序 **start()** 这个线程。

- 就绪状态:

  当线程对象调用了start()方法之后，该线程就进入就绪状态。就绪状态的线程处于就绪队列中，要**等待JVM里**线程调度器的调度。

- 运行状态:

  如果就绪状态的线程获取 CPU 资源，就可以执行 **run()**，此时线程便处于运行状态。处于运行状态的线程最为复杂，它可以变为阻塞状态、就绪状态和死亡状态。

- 阻塞状态:

  如果一个线程执行了sleep（睡眠）、suspend（挂起）等方法，失去所占用资源之后，该线程就从运行状态进入阻塞状态。在睡眠时间已到或获得设备资源后可以重新进入就绪状态。可以分为三种：

  - 等待阻塞：运行状态中的线程执行 wait() 方法，使线程进入到等待阻塞状态。
  - 同步阻塞：线程在获取 synchronized 同步锁失败(因为同步锁被其他线程占用)。
  - 其他阻塞：通过调用线程的 sleep() 或 join() 发出了 I/O 请求时，线程就会进入到阻塞状态。当sleep() 状态超时，join() 等待线程终止或超时，或者 I/O 处理完毕，线程重新转入就绪状态。

- 死亡状态:

  一个运行状态的线程完成任务或者其他终止条件发生时，该线程就切换到终止状态。

  

### 10.volatile

- 可见性（变量的）

  在多线程环境下，某个共享变量如果被其中一个线程给修改了，其他线程能够立即知道这个共享变量已经被修改了，当其他线程要读取这个变量的时候，最终会去内存中读取，而不是从自己的**工作空间**中读取。

- 有序性（代码的有序性）

- 代码写好之后，虚拟机不一定会按照我们写的代码的顺序来执行，虚拟机在进行代码编译优化的时候，对于那些改变顺序之后不会对最终变量的值造成影响的代码，是有可能将他们进行重排序的。



##### volatile关键字是如何保证线程安全问题的

在多线程环境下，某个共享变量如果被其中一个线程给修改了，其他线程能够立即知道这个**共享变量**（堆，方法区）已经被修改了，当其他线程要读取这个变量的时候，最终会去**内存**中读取，而不是从自己的**工作空间**（线程自己的工作区）中读取

**变量**被声明为volatile，那么这个变量就具有了**可见性**的性质了



##### volatile真的能完全保证一个变量的线程安全吗？

答案是**否定**的。原因是因为Java里面的运算并非是**原子操作**。

##### 原子操作

**原子操作**：即一个操作或者多个操作 要么**全部执行并且执行的过程不会被任何因素打断，要么就都不执行**。
也就是说，处理器要嘛把这组操作全部执行完，中间不允许被其他操作所打断，要嘛这组操作不要执行。
刚才说Java里面的运行并非是原子操作。

##### 什么情况下volatile能够保证线程安全

- 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
- 变量不需要与其他状态变量共同参与不变约束。

### 11.系列化与反序列化

序列化:对象写入到磁盘或者其他介质中，这个过程就叫做序列化

反序列化:把已存在在磁盘或者其他介质中的对象，反序列化（读取）到内存中