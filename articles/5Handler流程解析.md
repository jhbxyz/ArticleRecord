# 再谈Android消息机制

**本文目标**

- Android消息机制全流程把握
- Handler、Looper、MessageQueue、Message、Thread的关系

**注意本文基于Android-28**

**将从四个方面讲解Android消息机制**

* Handler的无参数构造方法
* Handler的sendMessage方法
* Handler的post方法
* Handler的构造方法：Handler(Callback callback) 

### 1.Handler无参数构造方法

举一个简单的例子，作为入口，例子中的Handler使用，会有内存泄漏问题（解决的话，可以派生子类，持有Activity的弱引用）。请注意，这里仅仅是举例

```java
public class DemoActivity extends AppCompatActivity {
  //1 构造方法
    private Handler mHanler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        new Thread(() -> {
            Message message = Message.obtain();
            message.obj = "Hanler.sendMessage方法"
              //2 Hanler的sendMessage方法
            mHanler.sendMessage(message);
        }).start();
    }
}
```

* Handler的构造方法
* Hanler的sendMessage方法

**先看Hanler构造方法**

```java
//Handler.java
public Handler() {
     this(null, false);

public Handler(Callback callback, boolean async) {
     mLooper = Looper.myLooper();//1
     if (mLooper == null) {
         throw new RuntimeException(
             "Can't create handler inside thread " + Thread.currentThread()
                     + " that has not called Looper.prepare()");
     }
     //2
     mQueue = mLooper.mQueue;
     mCallback = callback;//3 这个参数在稍后分析
     mAsynchronous = async;//4 是否是异步，默认发的消息是同步消息
 }
```

* 给mLooper赋值，首先获取Looper对象，为null则抛异常
* 给mQueue赋值
* mCallback会在第三个方面，分析Handler的构造方法：**Handler(Callback callback)** 中说明
* 这里的**异步不是线程**，跟同步屏障机制有关系（有兴趣的自己看下，View的绘制的过程）

**看一下Looper.myLooper()方法，是怎么获取Looper对象的**

```java
//Looper.java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();//1 ThreadLocal
}
```

从ThreadLocal中get了一个，既然有get肯定有set方法，在Looper类中找一下。

```java
//Looper.java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));//1
}
```

* 可以看到是调用的prepare这个静态方法，初始化了Looper并保存到ThreadLocal中

但是这个方法是什么时候调用了呢？我直接new Handler()，没调用这个方法，程序会崩溃的啊！

这就要说到咱们**程序的入口**方法了。

```java
//ActivityThread.java
public static void main(String[] args) {

    Looper.prepareMainLooper();//1

    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }

    Looper.loop();//2

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

* 给主线程消息循环，初始化Looper，所以在主线程new Handler()并不会报错
* 调用 Looper.loop();让消息循环起来

```java
//Looper.java
private static Looper sMainLooper; 
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

public static void prepareMainLooper() {
    prepare(false);//1
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();//2
    }
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));//3
}
```

* 调用的也是 prepare(false)方法，只是参数为false，这个参数是不允许Looper退出的
* 主线程Looper比较特殊，给sMainLooper赋值，主线程只会有一个Looper不然会抛异常（应该确切的说是每一个线程对应一个Looper）
* 初始化了Looper并保存到ThreadLocal中

**看一下ThreadLocal.get()和ThreadLocal.set()方法吧！**

```java
//ThreadLocal.java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

public void set(T value) {
    Thread t = Thread.currentThread();//1
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

* 这里Looper和先Thread相关联了，并且一个Thread对应一个Looper

在sThreadLocal.set(new Looper(quitAllowed))方法中，初始化了Looper

**再看一个Looper的构造方法**

```java
//Looper.java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);//1
    mThread = Thread.currentThread();//2
}
```

* 初始化了MessageQueue
* 记录了Thread

看到这儿，我们初始化了，Handler、Looper、MessageQueue。

**总结**

* 1.初始化Handler需要Looper支撑，主线程默认已经初始化了Looper
* 2.初始化Looper的时候初始化了MessageQueue
* 3.每个线程都是从ThreadLocal中获取，只会初始化一次Looper，不然会报错
* 4.所以Looper、MessageQueue、Thread一一对应

以上，Handler的无参数的构造方法分析完了，也知道了Looper、MessageQueue、Thread的对应关系，接下来调用Handle的sendMessage，看看都做了什么吧！

### 2.Handler的sendMessage方法

Handler有很多sendXXX方法，但是最终都会调用到enqueueMessage方法

```java
//Handler.java
public final boolean sendMessage(Message msg)
{
    return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;//1
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```
* 拿到了MessageQueue对象，这个对象是在上面有提到过，在Handler初始化的时候，和Looper，MessageQueue都是一一绑定的。

以上就是方法的层面调用，最终调用到 enqueueMessage(queue, msg, uptimeMillis);方法，我们来看下这个方法。

```java
//Handler.java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;//1 重点
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);//2
}
```

```java
//Message.java
public final class Message implements Parcelable {
/*package*/ Handler target;//1
}
```

* 给msg.target 赋值，这个target给Message的**成员变量**，是一个Handler，这是一个重点
* 调用MessageQueue的enqueueMessage方法

**MessageQueue的enqueueMessage方法**

```java
//MessageQueue.java
boolean enqueueMessage(Message msg, long when) {

    synchronized (this) {

        msg.markInUse();//1 标记Message使用
        msg.when = when;
        Message p = mMessages;//2 链表指针赋值给p
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {

            msg.next = p;
            mMessages = msg;//链表的头部赋值
            needWake = mBlocked;
        } else {

            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
          //不断遍历消息队列，根据when的比较找到合适的插入Message的位置。
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; 
            prev.next = msg;
        }

        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```
* 消息入队的操作，根据when来排序
* MessageQueue赋值Message的存和取

那么是在什么时候取数据呢，这就要用到

**Looper的loop方法了**

```java
//Looper.java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;


    for (;;) {
      //1 调用MessageQueue的next取出Message
        Message msg = queue.next(); // might block
       // 2 调用Looper的quit方法时 msg为null
        if (msg == null) {
           
            return;
        }
      
        final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
        final long dispatchEnd;
        try {
          // 3 关键方法,交给 Handler 处理
            msg.target.dispatchMessage(msg);
            dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
    
      

        msg.recycleUnchecked();//4 回收Message掉Pool中，复用
        }
}
```

主要看一下MessageQueue的next方法

```java
//MessageQueue.java
Message next() {

    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            //获取MessageQueue的链表表头的第一个元素
            Message msg = mMessages;
            //判断Message是否是障栅，如果是则执行循环，拦截所有同步消息，直到取到第一个异步消息为止
            if (msg != null && msg.target == null) {
                //如果能进入这个if，则表面MessageQueue的第一个元素就是障栅(barrier)
                //循环遍历出第一个异步消息，这段代码可以看出障栅会拦截所有同步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                    //如果msg==null或者msg是异步消息则退出循环，msg==null则意味着已经循环结束
                } while (msg != null && !msg.isAsynchronous());
            }

            if (msg != null) {
                if (now < msg.when) {
                    //
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    
                    mBlocked = false;
                    if (prevMsg != null) {
                        //上一个元素的next(越过自己)直接指向下一个元素
                        prevMsg.next = msg.next;
                    } else {
                        //如果没有上一个元素，则说明是消息队列中的头元素，直接让第二个元素变成头元素
                        mMessages = msg.next;
                    }
                    // 因为要取出msg，所以msg的next不能指向链表的任何元素，所以next要置为null
                    msg.next = null;

                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }
        }

        
    }
}
```

接下来代码会调用到 msg.target.dispatchMessage(msg);

* msg.target其实是我们发消息的Handler

**看一下Handler的dispatchMessage方法**

```java
//Handler.java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {//1
        handleCallback(msg);
    } else {
        if (mCallback != null) {//2
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);//3
    }
}
```

* msg.callback我们没有赋值，这个是Handler的postXXX方法赋值的
* mCallback也没有赋值，这是Handler的构造赋值的
* 所有走到了注释3：handleMessage方法，也就是我们一般写的方法

至此，Handler的sendMessage方法分析完了整个流程，也说明重写handleMessage方法的原因，接下来分析Handler的post方法

### 3.Handler的post方法

```java
//Handler.java
public final boolean post(Runnable r)
{
   return  sendMessageDelayed(getPostMessage(r), 0);//1
}

public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);//2
}
```
* 可以看到postXXX方法最终调用的也是sendXXX方法
* 最终都会调用到enqueueMessage方法，然后整个流程和sendXXX方法一致了。

看一下getPostMessage(r)方法，这个方法很关键

**Handler的getPostMessage**

```java
//Handler.java
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;//1
    return m;
}
```

* 把Runnable赋值给Message的callback成员变量
* 返回一个复用的Message

这个callback是干啥的？再看一下dispatchMessage方法

```java
//Handler.java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {//1
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

* 可见，使用post方法的时候，直接走的是 handleCallback(msg);方法

```java
//Message.java
private static void handleCallback(Message message) {
    message.callback.run();
}
```

* 直接调用的是Runnable的run()方法

### 4.Handler的构造方法：Handler(Callback callback) 

```java
//Handler.java
public Handler(Callback callback) {
    this(callback, false);
}

public Handler(Callback callback, boolean async) {

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;//1
    mAsynchronous = async;
}
```

* 这个构造会给mCallback赋值

这个mCallback是咋用的？再看一下dispatchMessage方法

```java
//Handler.java
public interface Callback {
    public boolean handleMessage(Message msg);
}
```

```java
//Handler.java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {//1
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

- 还是调用了handleMessage方法

至此，我们已经分析完了，Handle的构造方法，带Callback的构造方法，sendXXX方法，postXXX方法，把握了Android消息机制的整体流程。

