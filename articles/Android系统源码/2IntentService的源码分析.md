

# IntentService的源码分析

IntentService 是自己维护了一个线程，来执行耗时的操作，然后里面封装了HandlerThread，能够方便在**子线程**创建Handler。

IntentService 继承自 Service 用来处理**异步请求**的一个基类，通过startService发送请求，IntentService就被启动，然后会在一个工作线程中处理传递过来的Intent，当任务结束后就会**自动停止**服务。

> 开启一个后台线程，是一个后台服务，进程优先级高，不容易被杀死
>
> 进行异步请求
>
> 自动停止服务，每次只能处理一个任务，适用于请求数较少的情况

**疑惑**

* IntentService是如何开启一个工作线程的？
* 如何自动停止服务的？

### 1.IntentService 基本用法

```java
public class MyIntentService extends IntentService {

    public MyIntentService(String name) {
      //给线程定义名称的
        super("MyIntentService");
    }

    @Override
    protected void onHandleIntent(@Nullable Intent intent) {
        //后台处理任务
    }
}
```

我们在 `onHandleIntent` 方法中来处理，耗时操作。

### 2.IntentService 的 onHandleIntent方法

先看onHandleIntent 在 IntentService 类中 的调用过程

```java
//IntentService.java
protected abstract void onHandleIntent(@Nullable Intent intent);
```

是一个抽象方法

```java
//IntentService.java
private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
      //1
        onHandleIntent((Intent)msg.obj);
      //2
        stopSelf(msg.arg1);
    }
}
```

注释 1：在 ServiceHandler 的 `handleMessage`方法中调用了 `onHandleIntent` 方法

注释 2：调用了 `stopSelf`方法，

>  `stopSelf`方法，是 Service 自己的方法，用来停止自己的方法
>
> 所以在 当任务结束后，也就是 `onHandleIntent` 方法 执行完毕后，就会**自动停止**服务。

### 3.ServiceHandler 的初始化过程

在IntentService启动时，会最先调用 onCreate 方法

```java
//IntentService.java

private volatile Looper mServiceLooper;
private volatile ServiceHandler mServiceHandler;

@Override
public void onCreate() {
    super.onCreate();
  //1
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
  //2
    thread.start();
  //3
    mServiceLooper = thread.getLooper();
  //4
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
```

注释 1：初始化了 HandlerThread对象

注释 2：启动了HandlerThread这个子线程，并实例化 Looper

注释 3：获取了HandlerThread 这个子线程的 Looper 对象

注释 4：通过这个 Looper 对象实例化一个ServiceHandler

> 通过上面的几个步骤ServiceHandler的实例化过程就结束了
>
> ServiceHandler 的 `handleMessage`方法，会在 ServiceHandler这个 Handler 调用 `sendMessage` 方法时回调

我们看下 ServiceHandler 的 sendMessage 方法调用时机

### 4.ServiceHandler 的 sendMessage 方法调用时机

在IntentService启动时，会最先调用 onCreate 方法，然后就会调用 onStartCommand 方法

```java
//IntentService.java
@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
  //1
    onStart(intent, startId);
    return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}
```

注释 1：调用了 `onStart`方法

```java
//IntentService.java
@Override
public void onStart(@Nullable Intent intent, int startId) {
  //1
    Message msg = mServiceHandler.obtainMessage();
  //2
    msg.arg1 = startId;
  //3
    msg.obj = intent;
  //4
    mServiceHandler.sendMessage(msg);
}
```

注释 1：获取一个 Message 对象

注释 2：当前服务的标志，用来停止服务的

注释 3：把当前的 Intent 对象赋值给 msg.obj

注释 4：调用 mServiceHandler.sendMessage(msg) 方法

到现在，整个流程就完整了，在 IntentService `onCreate` 方法时，会实例化 mServiceHandler对象，在执行`onStartCommand`方法时，会进而调用 `onStart`方法，来发送消息，然后再处理消息

### 5.IntentService 的 onDestroy 方法

```java
//IntentService.java
@Override
public void onDestroy() {
    mServiceLooper.quit();
}
```

及时退出 Looper

### 6.总结

#### 6.1.如何开启一个子线程处理任务的？

* 在 IntentService 的 `onCreate` 方法中，实例化了一个 HandlerThread 对象，并通过 `start`方法，开启了这个子线程
* HandlerThread 在 `run`方法中实例化了，Looper 这个子线程对象
* 通过 Looper 这个子线程对象，实例化了 ServiceHandler 对象
* 在 `onStartCommand`方法总通过调用 `onStart`方法，把 Intent 这个实例包装成 Message
* 通过 ServiceHandler 的 `sendMessage` 方法来发送消息
* 在ServiceHandler的 `handleMessage`方法中调用 `onHandleIntent` 方法，在子线程处理任务

#### 6.2.如何自动停止服务的？

在ServiceHandler的 `handleMessage`方法中调用 `onHandleIntent` 方法，执行完毕后，会调用 `stopSelf`方法，来自动停止服务。

#### 6.3.注意的点

> * IntentService 的 `onCreate`方法只会执行一次，只会创建一个工作线程
>
> * 多次调用 `startService` 方法来开启一个IntentService，其 `onStartCommand` 方法也会执行多次，任务按顺序执行。