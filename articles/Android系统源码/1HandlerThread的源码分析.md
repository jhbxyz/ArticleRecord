# HandlerThread 的基础使用和源码分析

HandlerThread的源码并不多，总共才 100 多行，是一个继承了 Thread 的类，说明是一个线程，内部包装了 Looper 对象，Looper 中持有一个 MessageQueue 对象，底层其实是通过 `MessageQueue`来处理的

通过消息队列来重复使用当前线程，节省系统资源开销，但是一个单线程的任务，一旦队列中有某个任务执行时间过长，那么就会导致后续的任务都会被延迟处理。

### 1.HandlerThread 的使用

```java
public class HandlerThreadActivity extends AppCompatActivity {

    private HandlerThread handlerThread;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        handlerThread = new HandlerThread("后台线程-1");
        handlerThread.start();

        Handler handler = new Handler(handlerThread.getLooper()) {
            @Override
            public void handleMessage(@NonNull Message msg) {
                super.handleMessage(msg);
                System.out.println("当前线程是: " + Thread.currentThread().getName());
            }
        };
        handler.sendEmptyMessage(0x1);
    }

    @Override
    protected void onDestroy() {
        handlerThread.quit();
        super.onDestroy();
    }
}
```

> 打印结果：System.out: 当前线程是: 后台线程-1

* 通过HandlerThread的 `getLooper`方法来获取一个 Looper 对象，来实例化一个 Handler 对象
* 发送一个空消息

* 在`handleMessage`方法中打印当前的线程名

  > 通过HandlerThread 的 `getLooper`方法，获取 Looper 来指定 `handleMessage` 方法所在的线程，所以也就是子线程

* 在Activity`onDestroy`方法时，通过调用 `quit` 或 `quitSafely`退出循环

### 2.HandlerThread的源码分析

HandlerThread是一个 Thread，当调用 `start`方法时，执行的是 run 方法，我们就看HandlerThread的 `run`方法

#### 2.1.HandlerThread的 run 方法

```java
//HandlerThread.java
@Override
public void run() {
    mTid = Process.myTid();
  //1
    Looper.prepare();
    synchronized (this) {
      //2
        mLooper = Looper.myLooper();
      //3
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
  //4
    onLooperPrepared();
  //5
    Looper.loop();
    mTid = -1;
}
```

注释 1：当前线程实例化一个 Looper 对象，存到 ThreadLocal 里面

注释 2：获取当前线程的 Looper 对象，赋值给 mLooper

注释 3：通知正在 wait 的线程，去抢夺时间片，然后执行

注释 4：一个后台运行的方法，在 Looper 实例化后调用

注释 5：使 Looper 循环取来，去取消息

#### 2.2.HandlerThread 的 getLooper 方法

```java
//HandlerThread.java
public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }
    
    // If the thread has been started, wait until the looper has been created.
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
              //1
                wait();
            } catch (InterruptedException e) {
            }
        }
    }
    return mLooper;
}
```

注释 1：同步代码块中，判断线程是否活着，如果 mLooper 未初始化，调用 `wait`方法释放锁，等待 `notify`

> HandlerThread的 `run` 方法中，实例化 Looper 后，调用了 `notifyAll`来通知所有线程，争夺锁，获取Looper 对象

#### 2.2.HandlerThread 的 quit 方法

```java
//HandlerThread.java
public boolean quit() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quit();
        return true;
    }
    return false;
}
```

实际调用的是 Looper 的 `quit`方法，使消息循环停止。

#### 2.3.HandlerThread 的 getThreadHandler 方法

HandlerThread内部还有一个 Handler 实例，不过是被 @hide 了，也是通过当前线程的 Looper 创建的，具体使用会在 IntentServece 中提现出来。

```java
//HandlerThread.java

/**
 * @return a shared {@link Handler} associated with this thread
 * @hide
 */
@NonNull
public Handler getThreadHandler() {
    if (mHandler == null) {
        mHandler = new Handler(getLooper());
    }
    return mHandler;
}
```

### 3.总结

* HandlerThread 是一个 Thread 的子类，是一个线程

* 初始化 HandlerThread，调用 `start`方法，开启一个后台线程

* 通过 HandlerThread 的实例获取一个 Looper 对象，进而实例化一个 Handler 对象，参数就是这个 Looper

* 通过 Handler 的一系列的 `sendXXX` 方法，`postXXX` 方法来发送消息

* 最后通过 Handler 的 handleMessage 中处理这些任务

  > 这些任务是存在 MessageQueue 中，是排队顺序执行的

* 在不用 HandlerThread 的时候，及时调用 HandlerThread 的 `quit` 方法，结束循环，节省资源。

* 通过 HandlerThread 可以获得Looper 对象，这个 Looper 就是可以指定我们在哪个线程去执行任务的

