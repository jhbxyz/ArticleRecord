# Java线程线程池基础

- Thread
- Runnable
- 线程池
- Callable
- 线程的生命周期
- 线程间的通信

在Android中在很多情况下为了使APP更加流程地运行，我们不可能将很多事情都放在主线程上执行，需要开启子线程来进行网络请求，读取文件、数据库，或者是操作图片等。

### 1.多线程的实现方式

#### 1.1.继承Thread 类

写一个类继承Thread，重写 `run`方法，调用 `start`方法，我们在 `run`方法中的代码就在后台运行了，具体是由操作系统执行切线程操作，`run`方法由 JVM 调用。

```java
public class ThreadDemo {

    public static void main(String[] args) {
        new MyThread().start();
    }

    static class MyThread extends Thread {
        @Override
        public void run() {
            super.run();
            // do something
        }
    }
}
```

#### 1.2.实现 Runnable 接口

Runnable**只是一个接口**，所以单看这个**接口它和线程毫无关系**，Thread调用了Runnable接口中的方法用来在线程中执行任务

Java是单继承但可以调用多个接口，所以看起来Runnable更加好一些，在实际使用中，我们其实用的是**线程池**。

```java
public class ThreadDemo {

    public static void main(String[] args) {
        MyRunnable myRunnable = new MyRunnable();
        new Thread(myRunnable).start();
    }
    static class MyRunnable implements Runnable {
        @Override
        public void run() {
            // do something
        }
    }
}
```

#### 1.3.run() 方法和start() 方法的区别

- run()方法只是调用了 Thread 实例的 run() 方法而已，它仍然运行在**主线程**上
- start() 方法会开辟一个新的线程，在新的线程上调用 run() 方法，此时它运行在新的线程上，由 **JVM 调用**。

### 2.Java线程池

一个系统而言，创建、销毁、调度线程的过程是需要开销的，所以我们并不能无限量地开启线程，所以就需要管理线程。

#### 2.1.线程池的优点

- **重用**线程池中的线程，避免频繁地创建和销毁线程带来的性能消耗；
- 有效**控制**线程的最大并发数量，防止线程过大导致抢占资源造成系统阻塞；
- 可以对线程进行一定地**管理**。

#### 2.2.ThreadPoolExecutor

ExecutorService 是最初的线程池接口，ThreadPoolExecutor 类是对线程池的具体实现

```java
public ThreadPoolExecutor(int corePoolSize,//核心线程的数量
                          int maximumPoolSize,//最大线程数
                          long keepAliveTime,//非核心线程的超时时长
                          TimeUnit unit,//枚举时间单位
                          BlockingQueue<Runnable> workQueue,//线程池中的任务队列
                          ThreadFactory threadFactory,//线程工厂
                          //超出限制，拒绝执行
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

- corePoolSize 核心线程的数量，**默认一直存活**，避免了一般情况下CPU创建和销毁线程带来的开销，**allowCoreThreadTimeOut**属性设置为true，那么闲置的核心线程就会有超时策略，这个时间由keepAliveTime来设定；allowCoreThreadTimeOut默认为false，核心线程没有超时时间。
- maximumPoolSize 最大线程数，最大线程数=**核心线程+非核心线程**，非核心线程只有当核心线程不够用且线程池有空余时才会被创建，执行完任务后非核心线程会被销毁，数量超过最大线程数时其它任务可能就会被阻塞
- keepAliveTime **非核心线程**的超时时长，当执行时间超过这个时间时，非核心线程就会被回收。当allowCoreThreadTimeOut设置为true时，此属性也作用在核心线程上。
- unit，枚举时间单位，TimeUnit。
- workQueue，线程池中的任务队列，我们提交给线程池的runnable会被存储在这个对象上。

线程池的分配遵循这样的规则：

- 当线程池中的核心线程数量未达到最大线程数时，启动一个核心线程去执行任务；
- 如果线程池中的核心线程数量达到最大线程数时，那么任务会被插入到任务队列中排队等待执行；
- 如果在上一步骤中任务队列已满但是线程池中线程数量未达到限定线程总数，那么启动一个非核心线程来处理任务；
- 如果上一步骤中线程数量达到了限定线程总量，那么线程池则拒绝执行该任务，且ThreadPoolExecutor会调用RejectedtionHandler的rejectedExecution方法来通知调用者。

#### 2.3.线程池的分类

都直接或者间接通过ThreadPoolExecutor来实现自己的功能，当然你也可以自己去实现一个线程池

- FixedThreadPool
- CachedThreadPool
- ScheduledThreadPool
- SingleThreadExecutor

##### 2.3.1.FixedThreadPool

Executors的newFixedThreadPool()方法创建，它是个线程数量固定的线程池，该线程池的线程全部为核心线程，它们没有超时机制且排队任务队列无限制，因为全都是核心线程，所以响应较快，且不用担心线程会被回收。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

##### 2.3.2.CachedThreadPool

Executors的newCachedThreadPool()方法来创建，它是一个数量无限多的线程池，它所有的线程都是非核心线程，当有新任务来时如果没有空闲的线程则直接创建新的线程不会去排队而直接执行，并且超时时间都是60s，所以此线程池适合执行大量耗时小的任务。由于设置了超时时间为60s，所以当线程空闲一定时间时就会被系统回收，所以理论上该线程池不会有占用系统资源的无用线程。

```java
 public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

##### 2.3.3.ScheduledThreadPool

Executors的newScheduledThreadPool()方法来创建，ScheduledThreadPool 线程池像是上两种的合体，它有数量固定的核心线程，且有数量无限多的非核心线程，但是它的非核心线程超时时间是0s，所以非核心线程一旦空闲立马就会被回收。这类线程池适合用于执行定时任务和固定周期的重复任务。

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```

##### 2.3.4.SingleThreadExecutor

Executors的newSingleThreadExecutor()方法来创建，它内部只有一个核心线程，它确保所有任务进来都要排队按顺序执行。它的意义在于，统一所有的外界任务到同一线程中，让调用者可以忽略线程同步问题。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

#### 2.4.线程池一般用法

- shutDown()，关闭线程池，需要执行完已提交的任务；
- shutDownNow()，关闭线程池，并尝试结束已提交的任务；
- allowCoreThreadTimeOut(boolen)，允许核心线程闲置超时回收；
- execute()，提交任务无返回值；
- submit()，提交任务有返回值；

### 3.Callable 和 FutureTask

**Runnable 和 Callable** 都代表那些要在不同的线程中执行的任务，Thread类只支持Runnable接口，所以就需要用到 FutureTask

它们的主要区别是 Callable 的 call() 方法可以返回值和抛出异常，而 Runnable 的 run() 方法没有这些功能。Callable 可以返回装载有计算结果的 Future 对象。

FutureTask 实现了 Runnable 和 Future，所以兼顾两者优点，既可以在 Thread 中使用，又可以在 ExecutorService 中使用。

#### 3.1直接在 Thread 中使用

```java
FutureTask<String> task = new FutureTask<String>(new Callable<String>() {
    @Override
    public String call() throws Exception {
        return "Anything";
    }
});

Thread thread = new Thread(task);
thread.start();
```

#### 3.2在线程池中使用

```java
FutureTask<String> task = new FutureTask<String>(new Callable<String>() {
    @Override
    public String call() throws Exception {
        return "Anything";
    }
});

Executors.newCachedThreadPool().submit(task);

try {
  	//调用 get 方法来获取返回值，注意，这是个阻塞方法
    String result = task.get();
    System.out.println("result" + result);
} catch (ExecutionException e) {
    e.printStackTrace();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

#### 3.3.FutureTask的好处

- FutureTask 是为了弥补 Thread 的不足而设计的
- 准确地知道线程什么时候执行完成并获得到线程执行完成后返回的结果
- FutureTask 是一种可以取消的异步的计算任务，它的计算是通过 Callable 实现的，它等价于可以携带结果的 Runnable
- 三个状态：等待、运行和完成

#### 3.4 Callable 和 Runnable

它们的主要区别是 Callable 的 call() 方法可以**返回值和抛出异常**，而 Runnable 的 run() 方法没有这些功能。Callable 可以返回装载有计算结果的 **Future 对象**

- Callable 接口下的方法是 call()，Runnable 接口的方法是 run()；
- Callable 的任务执行后可返回值，而 Runnable 的任务是不能返回值的；
- call() 方法可以抛出异常，run()方法不可以的；
- 运行 Callable 任务可以拿到一个 Future 对象，表示**异步计算的结果**。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过 Future 对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果；

### 4.线程的生命周期

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

### 5.线程间的通信

#### 5.1.Java中的 wait、notify、notifyAll方法

- Object类的3个本地方法
- 调用这3个方法的时候，当前线程必须获得这个对象的锁
- 调用一个Object的wait与notify/notifyAll的时候，必须保证调用代码对该Object是同步的

wait：线程自动释放其占有的对象锁，并等待notify
notify：唤醒一个正在wait当前对象锁的线程，并让它拿到对象锁
notifyAll：唤醒所有正在wait前对象锁的线程

注意：**notify**是本地方法，具体唤醒哪一个线程由虚拟机控制；notifyAll后并不是所有的线程都能马上往下执行，它们**只是跳出了wait状态**，接下来它们还会是竞争对象锁。





### 参考文章

[Android 线程和线程池一篇就够了](https://juejin.im/entry/6844903480193187854)