# AsyncTask 的基础用法和源码分析

AsyncTask 内部封装了线程池和 Handler，可以让我们轻松做到在后台进行计算并且把结果及时更新到UI上，通过 AsyncTask 可以不用自己写 Thread 和 Handler，这是一种简便用法。

> 虽然在 Android R 也就是 API 30，AsyncTask 已经被标记为过时，但这个丝毫不影响我们学习这个类

你是否在使用AsyncTask 中有下面的疑惑

- AsyncTask 的整个运行的流程是什么？
- AsyncTask 各个方法是什么时候回调的？
- AsyncTask 的串行并行是怎么实现的？
- 底层后台的线程池是怎样的？

本文整体流程是，写一个例子来简单使用 AsyncTask 模拟后台请求UI更新，然后这个例子，来作为源码分析的入口，通过源码逐步解开我们的疑惑

### 1.AsyncTask的基础用法

模拟一个后台请求，输出结果，更新界面，最后来分别解释各个方法的含义

```java
public class AsyncTaskActivity extends AppCompatActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
      //1
        MyAsyncTask asyncTask = new MyAsyncTask();
      //2
        asyncTask.execute();
    }

    class MyAsyncTask extends AsyncTask<Void, Integer, Integer> {
        private int value;

        @Override
        protected void onPreExecute() {
            super.onPreExecute();
            //开始前
            value = 0;
        }

        @Override
        protected Integer doInBackground(Void... voids) {
            //模拟耗时
            for (int i = 0; i < 100; i++) {
                value++;
                //通知更新
                publishProgress(value);
            }
            return value;
        }

        @Override
        protected void onProgressUpdate(Integer... values) {
            super.onProgressUpdate(values);
            //更新进度
            System.out.println("当前进度为: " + values[0]);
        }

        @Override
        protected void onPostExecute(Integer result) {
            super.onPostExecute(result);
            //最终结果
            System.out.println("结果为: " + result);
        }
      
        @Override
        protected void onCancelled() {
            super.onCancelled();
        }

    }
}
```

**日志结果**

```log
System.out: 当前进度为: 1
...
System.out: 当前进度为: 100
System.out: 结果为: 100
```

注释 1：实例化 MyAsyncTask 对象，AsyncTask的**构造方法**是重点，下面会详细看！

注释 2：调用 MyAsyncTask 的 `execute`方法，开始执行。

说明一下 MyAsyncTask 类的各个回调方法

* onPreExecute：开始前执行，做一些初始化操作
* doInBackground：后台运行，执行耗时操作
* onProgressUpdate：更新进度的回调方法，通过 `publishProgress`配合使用
* onPostExecute：结果回调，处理操作UI
* onCancelled：用户调用取消时

> 除了`doInBackground`方法，运行在子线程，其余方法全都运行在主线程。

以上就是一个简单使用的例子，接下来进行源码分析

上面代码中初始化了一个 AsyncTask 对象，执行了 `execute` 方法， 先看 `asyncTask.execute();`方法，这个是执行的方法，我们以此为分析的入口，等一会儿再看 AsyncTask 构造方法

### 2.AsyncTask 的 execute 方法

```java
//AsyncTask.java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
  //1
    return executeOnExecutor(sDefaultExecutor, params);
}

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
  //2
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }
  //3
    mStatus = Status.RUNNING;
  //4
    onPreExecute();
  //5
    mWorker.mParams = params;
  //6
    exec.execute(mFuture);

    return this;
}
```

注释 1：`execute` 方法调用 `executeOnExecutor` 方法，其中有一个 sDefaultExecutor，来看下这个变量

> ```java
> //AsyncTask.java
> private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
> 
> public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
> ```
>
> 是一个线程池 SerialExecutor，用来执行的后台任务的，是一个串行的线程池，任务挨个执行，下面会具体分析

注释 2：判断mStatus的状态，如果是正在执行，抛异常，所以只能执行一次

> 如果我们再次执行一遍这个任务，那么里面的数据肯定不是我们所期望的结果

注释 3：mStatus 的状态赋值为 Status.RUNNING

注释 4：回调 `onPreExecute` 方法在开始执行后台任务之前执行，所以在主线程

注释 5：给 mWorker 这个对象的 mParams 变量赋值，接下来会用到

> 需要看下 mWorker 是什么，以及在什么时候初始化的

注释 6：调用 sDefaultExecutor 线程池的 `execute`方法，执行 mFuture的内容

> 需要看下 mFuture 是什么，以及在什么时候初始化的

### 3.AsyncTask 的构造方法

我们实例化 MyAsyncTask 对象的时候，调用的是无参的构造方法。

```java
//AsyncTask.java
public AsyncTask() {
    this((Looper) null);
}

```

```java
//AsyncTask.java
public AsyncTask(@Nullable Looper callbackLooper) {
  //1
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);
  //2
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
          //3
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
              //4
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
              //5
                postResult(result);
            }
            return result;
        }
    };
  //6
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
              //7
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}
```

注释 1：给 mHandler 赋值，因为 `callbackLooper == null`所以调用的是 `getMainHandler()`方法

> ```java
> //AsyncTask.java
> private static Handler getMainHandler() {
>     synchronized (AsyncTask.class) {
>         if (sHandler == null) {
>             sHandler = new InternalHandler(Looper.getMainLooper());
>         }
>         return sHandler;
>     }
> }
> ```
>
> 实例化了一个主线程的 Handler，也就是 InternalHandler 对象，这个会在后面细看它的 `handleMessage`方法

注释 2：给mWorker 赋值，实例化 WorkerRunnable 对象

> ```java
> //AsyncTask.java
> private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
>     Params[] mParams;
> }
> ```
>
> 是一个抽象类，实现了Callable接口
>
> 声明了一个 mParams 的成员变量，给这个变量赋值

注释 3：mTaskInvoked 是一个 原子布尔值，保证了，线程间的，同步性，可见性，把mTaskInvoked的值设为 true

注释 4：回调 `doInBackground` 方法，把结果赋值给 result

注释 5：调用 `postResult` 方法，把 result 传进去

> ```java
> //AsyncTask.java
> private Result postResult(Result result) {
>   //获取 mHandler 也就是主线程的 Handler，获取一个Message，把 AsyncTaskResult 传过去
>     Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
>             new AsyncTaskResult<Result>(this, result));
>   //发送消息，然后让 InternalHandler 来处理消息，当前的 what 值是 MESSAGE_POST_RESULT
>     message.sendToTarget();
>     return result;
> }
> 
> //获取的是 InternalHandler 对象
> private Handler getHandler() {
>     return mHandler;
> }
> ```
>
> ```java
> //AsyncTask.java
> private static class InternalHandler extends Handler {
>     public InternalHandler(Looper looper) {
>         super(looper);
>     }
> 
>     @Override
>     public void handleMessage(Message msg) {
>       //取出传递的结果
>         AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
>         switch (msg.what) {
>             // 根据 what 值，来调用
>             case MESSAGE_POST_RESULT:
>             //执行 AsyncTask 的 finish 方法
>                 result.mTask.finish(result.mData[0]);
>                 break;
>             case MESSAGE_POST_PROGRESS:
>                 result.mTask.onProgressUpdate(result.mData);
>                 break;
>         }
>     }
> }
> ```
>
> 看一下AsyncTaskResult这个类的结构
>
> ```java
> //AsyncTask.java
> private static class AsyncTaskResult<Data> {
>     final AsyncTask mTask;
>     final Data[] mData;
> 
>     AsyncTaskResult(AsyncTask task, Data... data) {
>         mTask = task;
>         mData = data;
>     }
> }
> ```
>
> result.mTask 是 AsyncTask对象，调用它的 `finish`方法
>
> ```java
> //AsyncTask.java
> private void finish(Result result) {
>     if (isCancelled()) {
>       //如果取消了，回调 onCancelled 方法
>         onCancelled(result);
>     } else {
>       //回调 onPostExecute 方法
>         onPostExecute(result);
>     }
>   //更新 mStatus的值为 Status.FINISHED
>     mStatus = Status.FINISHED;
> }
> ```
>
> 到现在，整个 AsyncTask 的各个回调方法，都被调用了
>
> * onPreExecute，主线程调用，在线程开启调用，做一些初始化工作，在 `doInBackground`方法之前调用
>
> * doInBackground 通过线程池调用，是运行在子线程的
> * onCancelled 通过主线程 Handler 的 `handleMessage`方法调用，运行在主线程
>
> * onPostExecute 通过主线程 Handler 的 `handleMessage`方法调用，运行在主线程
>
> 现在还差一个 `onProgressUpdate` 方法的调用时机，没有看，下面会提到。

注释 6：初始化 FutureTask 对象，并把 mWorker作为参数传递，然后通过线程池执行

注释 7：执行 `postResultIfNotInvoked`，来具体看下这个方法

> ```java
> //AsyncTask.java
> private final AtomicBoolean mTaskInvoked = new AtomicBoolean();
> 
> private void postResultIfNotInvoked(Result result) {
>   // 通过 mTaskInvoked的取值
>     final boolean wasTaskInvoked = mTaskInvoked.get();
>   //wasTaskInvoked为 false 表示没有执行 postResult方法
>     if (!wasTaskInvoked) {
>       //执行 postResult方法
>         postResult(result);
>     }
> }
> ```
>
> 在初始化 mWorker 对象时，被赋值为 true，所以此时不会再调用 `postResult`方法了。

### 4.AsyncTask 的 onProgressUpdate 方法的调用时机

`onProgressUpdate` 方法回调时，是通过 `publishProgress`方法

```java
//AsyncTask.java
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
```

通过 InternalHandler 发送了一个消息，Message 的 what 值是 MESSAGE_POST_PROGRESS，现在看下 InternalHandler 的 `handleMessage`方法

```java
//AsyncTask.java
private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }

    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
            //1
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```

注释 1：回调了 onProgressUpdate 方法

### 5.AsyncTask 的串行和并行

#### 5.1.写一个 AsyncTaskTest 来继承 AsyncTask 

```java
class AsyncTaskTest extends AsyncTask {
    private String taskName;

    public AsyncTaskTest(String taskName) {
        this.taskName = taskName;
    }

    @Override
    protected Object doInBackground(Object[] objects) {
      //睡眠 3 秒
        SystemClock.sleep(3000);
        return null;
    }

    @Override
    protected void onPostExecute(Object o) {
        super.onPostExecute(o);
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
      //打印当前时间
        System.out.println("taskName = " + taskName + "  时间: " + sdf.format(new Date()));

    }
}
```

#### 5.2.执行 `execute`方法，并看结果日志

```java
new AsyncTaskTest("AsyncTask1").execute();
new AsyncTaskTest("AsyncTask2").execute();
new AsyncTaskTest("AsyncTask3").execute();

日志结果
System.out: taskName = AsyncTask1  时间: 2020-09-15 15:37:00
System.out: taskName = AsyncTask2  时间: 2020-09-15 15:37:03
System.out: taskName = AsyncTask3  时间: 2020-09-15 15:37:06
```

> 执行 `execute`方法，结果是串行，每隔 3 秒执行一次

#### 5.3.执行 `executeOnExecutor` 方法

```java
new AsyncTaskTest("AsyncTask1").executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
new AsyncTaskTest("AsyncTask2").executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
new AsyncTaskTest("AsyncTask3").executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);

日志结果
System.out: taskName = AsyncTask1  时间: 2020-09-15 15:37:39
System.out: taskName = AsyncTask2  时间: 2020-09-15 15:37:39
System.out: taskName = AsyncTask3  时间: 2020-09-15 15:37:39
```

> 执行 `executeOnExecutor` 方法，是并行，同时执行

AsyncTask即可串行又可以并行，具体就看调用的方法是哪个，其实是线程池的原因。

看一下为什么是串行执行的？稍后再看下并行执行的原因

#### 5.4执行 execute 方法为什么是串行的

看下 `execute` 方法

```java
//AsyncTask.java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```

> 调用的也是 executeOnExecutor 方法，不过传递的线程池是  sDefaultExecutor

看一下这个 sDefaultExecutor 具体是什么

```java
//AsyncTask.java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                  //1
                    r.run();
                } finally {
                  //4
                    scheduleNext();
                }
            }
        });
      //2
        if (mActive == null) {
            scheduleNext();
        }
    }
  //3
    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}

```

> sDefaultExecutor 是一个静态的 volatile 修饰的，线程间可见的线程池
>
> 它在AsyncTask中是以常量的形式被使用的，因此在整个应用程序中的所有AsyncTask实例都会共用同一个SerialExecutor

注释 1：执行 Runnable 的 run 方法，其实执行的是 mFuture 这个 FutureTask 的 run 方法

> 把 Runnable 添加到 mTasks 这个队尾中去

注释 2：第一执行任务时，此时 mActive 等于空，所以执行 `scheduleNext`方法

注释 3：mTasks中从对头**取出并移除**一个 Runnable 赋值给 mActive ，如果不是空，才通过 THREAD_POOL_EXECUTOR 这个线程池执行这个 Runnable

注释 4：执行 `scheduleNext` 方法，从mTasks队列中取出的值不为空，才执行

#### 5.5.执行executeOnExecutor 方法为什么是并行的

主要是线程池不同，这里用到是 THREAD_POOL_EXECUTOR 这个线程池

```java
private static final int CORE_POOL_SIZE = 1;
private static final int MAXIMUM_POOL_SIZE = 20;
private static final int BACKUP_POOL_SIZE = 5;
private static final int KEEP_ALIVE_SECONDS = 3;

public static final Executor THREAD_POOL_EXECUTOR;

static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>(), sThreadFactory);
    threadPoolExecutor.setRejectedExecutionHandler(sRunOnSerialPolicy);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

* 核心线程池：1
* 最大线程池：20
* 超时时间：3 秒

> 在小于 20 个任务时，都会同时执行，当然你可以自己配置线程池

### 6.几个使用 AsyncTask 的问题

#### 6.1.为什么 AsyncTask 不适合执行长时间耗时操作？

* 首先AsyncTask是可以执行耗时操作的，因为底层是通过线程池执行任务，也就是一个后台线程

* AsyncTask 只是不适合执行长时间耗时操作，使用AsyncTask的目的是为了方便处理后台任务，快速更新 UI，操作简单，而且默认 `execute`方法启动，是串行的，如果其中几个AsyncTask后台任务执行过长，那后面的AsyncTask 处于排队中，等待时间太久

  > 可以使用 executeOnExecutor 方法，来并发执行任务

#### 6.2.AsyncTask一定要在UI线程执行吗

如果在子线程初始化 AsyncTask 并调用它的 `execute`方法

* `onPreExecute`方法在子线程执行，可能做一些跟 UI 相关的如：显示 Loading 对话框，会出错，子线程不能更新 UI

  > onPreExecute 在子线程执行的原因是：AsyncTask 在子线程调用它的 `execute`方法

* `doInBackground` 方法还是在子线程运行，这个并没有改变

* `onProgressUpdate`和 `onPostExecute` 和 `onCancelled` 方法在主线程运行，这个并没有改变，因为这两个方法是通过 Handler 处理的，进行了线程切换

### 7.总结

#### 7.1.AsyncTask 的整个运行的流程是什么？AsyncTask 各个方法是什么时候回调的？

* 初始化 AsyncTask 对象，执行 `execute`方法，进而执行 `executeOnExecutor`方法，采用默认的SerialExecutor线程池 ，是一个串行线程池
* 在执行 `executeOnExecutor`方法中回调 onPreExecute()方法，给 mWorker 赋值，并通过SerialExecutor线程池执行 mFuture中的代码
* 在 AsyncTask 构造中，初始化 mWorker 和 mFuture 对象，在执行 mFuture 这个 Runnable 时，执行 mWorker 这个Callable 的 `call` 方法，然后回调 `doInBackground`方法，最后执行 `postResult`方法
* 在 `postResult`中，通过 Handler 把当前的 AsyncTask 对象和结果发送到主线程，在 `handleMessage`中处理
* 最后调用的是 AsyncTask 的 `finish`方法，根据是否取消，分别执行 `onCancelled`和 `onPostExecute`方法

#### 7.2.AsyncTask 的串行并行是怎么实现的？

* 通过执行 `execute`方法，执行任务，采用默认的SerialExecutor线程池 ，是一个串行线程池。

* 通过执行 `executeOnExecutor`方法，可以采用 AsyncTask 提供的 THREAD_POOL_EXECUTOR线程池实现并行执行

  > 当然也可以自己配置线程池

#### 7.3.底层后台的线程池是怎样的？

> 具体看 5.4 和 5.5 小节

