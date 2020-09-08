# OkHttp请求的流程的梳理

本文为 [OkHttp](https://square.github.io/okhttp/) 的第一篇文章，主要是对整个请求的流程的梳理，对 OkHttp 整体有一个感性的认识。

> 本文基于 OkHttp 最新的 4.8.1版本进行源分析的，源码是 Kotlin 写的，做好准备。

**依赖**

```groovy
implementation 'com.squareup.okhttp3:okhttp:4.8.1'
```

**整体流程**

先写一个 Demo 作为源码分析的入口，首先分析异步请求的整体流程，然后再看同步请求的整体流程，从整体上把握 OkHttp 的请求流程

### 1.OkHttp 的基本使用

```java
class OkHttpActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        //1 请求的 Client
        val okHttpClient = OkHttpClient()
        //2 构造出一个 Request 对象
        val request = Request.Builder()
            //API 接口由 wanandroid.com 提供
            .url("https://wanandroid.com/wxarticle/chapters/json")
            .get()
            .build()
        //3 创建出一个执行的 Call 对象
        val call = okHttpClient.newCall(request)
        //4 异步执行请求
        call.enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {
                e.printStackTrace()
            }

            override fun onResponse(call: Call, response: Response) {
                "response.code = ${response.code}".log()
            }
        })
    }
}
```

以上就是 OkHttp 的最基本的用法，接口请求的是[鸿洋](https://juejin.im/user/1697301652584670)提供的[玩Android 开放API](https://wanandroid.com/blog/show/2)

注释 1：实例化一个OkHttpClient对象，用来配置请求的 interceptors（插值器）读写超时时间等配置，其实通过内部类 Builder 构建的，内部会初始化一个 Dispatcher 对象，下面会用到。

```kotlin
//OkHttpClient.kt
constructor() : this(Builder())
// OkHttpClient内部的Builder类，通过
class Builder constructor() {
  //调度器
	internal var dispatcher: Dispatcher = Dispatcher()
  //拦截器 list
	internal val interceptors: MutableList<Interceptor> = mutableListOf()
  //网络拦截器 list
	internal val networkInterceptors: MutableList<Interceptor> = mutableListOf()
	internal var eventListenerFactory: EventListener.Factory = EventListener.NONE.asFactory()
	internal var retryOnConnectionFailure = true
	internal var connectTimeout = 10_000
	internal var readTimeout = 10_000
	internal var writeTimeout = 10_000
}
```

注释 2：构建一个请求的对象

注释 3：通过OkHttpClient 创建出一个执行的 Call 对象，其实是一个 RealCall，这个是真正进行请求的

注释 4：调用 call 的 `enqueue`方法，执行请求。

> Call 有两种请求方式： `enqueue`异步请求方法和 `execute`同步方法
>
> 先看 `enqueue`异步请求的全过程
>
>  `execute`同步方法的过程，会在后面具体分析

### 2.源码分析

我们分析的入口就是 call 的 `enqueue`方法

```kotlin
//Call.kt
interface Call : Cloneable {
 	...
  fun enqueue(responseCallback: Callback)
	...
}
```

是一个接口，所以我们要看具体的 Call 对象是谁，`okHttpClient.newCall(request)`方法返回了一个 Call 对象

看一下这个方法

### 3.OkHttpClient 的 newCall 方法

```kotlin
//OkHttpClient.kt
/** Prepares the [request] to be executed at some point in the future. */
override fun newCall(request: Request): Call = RealCall(this, request, forWebSocket = false)
```

是一个 RealCall 对象，好的看一下 RealCall 的 `enqueue`方法

### 4.RealCall 的 enqueue 方法

```kotlin
//RealCall.kt
override fun enqueue(responseCallback: Callback) {
  check(executed.compareAndSet(false, true)) { "Already Executed" }

  //1
  callStart()
  //2
  client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```

注释 1：调用的是 EventListener 的 `callStart`方法，是一个全局监听请求过程的方法，不是重点

注释 2：调用 dispatcher 的 `enqueue`方法，传进去了 AsyncCall 对象和 responseCallback参数

> AsyncCall下面会具体看，responseCallback就是我们代码写的 CallBack，用来回调请求结果的。

### 5.Dispatcher 的 enqueue 方法

```kotlin
//Dispatcher.kt

/** Ready async calls in the order they'll be run. */
private val readyAsyncCalls = ArrayDeque<AsyncCall>()

internal fun enqueue(call: AsyncCall) {
  synchronized(this) {
    //1
    readyAsyncCalls.add(call)

    if (!call.call.forWebSocket) {
      val existingCall = findExistingCallWithHost(call.host)
      if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
    }
  }
  //2
  promoteAndExecute()
}
```

注释 1：把 AsyncCall 对象添加到 readyAsyncCalls 这个 list 中，这个集合下面会用到

注释 2：`promoteAndExecute` 推举并执行，这个方法是重点。

### 6.Dispatcher 的 promoteAndExecute 方法

```kotlin
//Dispatcher.kt
private fun promoteAndExecute(): Boolean {
  this.assertThreadDoesntHoldLock()
	//1
  val executableCalls = mutableListOf<AsyncCall>()
  val isRunning: Boolean
  synchronized(this) {
    //2
    val i = readyAsyncCalls.iterator()
    while (i.hasNext()) {
      val asyncCall = i.next()

      if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
      if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.

      i.remove()
      asyncCall.callsPerHost.incrementAndGet()
      //3
      executableCalls.add(asyncCall)
      runningAsyncCalls.add(asyncCall)
    }
    isRunning = runningCallsCount() > 0
  }
	//4
  for (i in 0 until executableCalls.size) {
    val asyncCall = executableCalls[i]
    //5
    asyncCall.executeOn(executorService)
  }

  return isRunning
}
```

注释 1：声明一个 AsyncCall 的集合

注释 2：遍历readyAsyncCalls这个集合

注释 3：取出每一个AsyncCall存到 executableCalls 这个集合中

注释 4：遍历 executableCalls 这个集合

注释 5：调用 asyncCall 的 `executeOn` 方法传入了参数executorService，这个是重点

> 其中executorService的为ExecutorService是一个线程池，用来执行后台任务的
>
> ```kotlin
> //Dispatcher.kt
> @get:Synchronized
> @get:JvmName("executorService") val executorService: ExecutorService
>   get() {
>     if (executorServiceOrNull == null) {
>       //自定义一个没有核心线程，非核心线程不限的，且在闲置的时候，一分钟后会被回收的线程池
>       executorServiceOrNull = ThreadPoolExecutor(0, Int.MAX_VALUE, 60, TimeUnit.SECONDS,
>           SynchronousQueue(), threadFactory("$okHttpName Dispatcher", false))
>     }
>     return executorServiceOrNull!!
>   }
> ```

### 7.AsyncCall 的 executeOn 方法

```kotlin
//AsyncCall.kt
fun executeOn(executorService: ExecutorService) {
  client.dispatcher.assertThreadDoesntHoldLock()

  var success = false
  try {
    //1
    executorService.execute(this)
    success = true
  } catch (e: RejectedExecutionException) {
    val ioException = InterruptedIOException("executor rejected")
    ioException.initCause(e)
    noMoreExchanges(ioException)
    //2
    responseCallback.onFailure(this@RealCall, ioException)
  } finally {
    if (!success) {
      client.dispatcher.finished(this) // This call is no longer running!
    }
  }
}
```

注释 1：把 AsyncCall 放到后台执行，那AsyncCall肯定一个Runnable 了，一会儿去看下它的 `run`方法做了什么

注释 2：线程池任务满了，拒绝执行，抛异常直接回调失败。

### 8.AsyncCall 的 run 方法

```kotlin
//RealCall.kt 的内部类
//AsyncCall.kt
internal inner class AsyncCall(
    private val responseCallback: Callback
  ) : Runnable {
    
    override fun run() {
      threadName("OkHttp ${redactedUrl()}") {
        var signalledCallback = false
        timeout.enter()
        try {
          //1
          val response = getResponseWithInterceptorChain()
          signalledCallback = true
          //2
          responseCallback.onResponse(this@RealCall, response)
        } catch (e: IOException) {
          if (signalledCallback) {
            // Do not signal the callback twice!
            Platform.get().log("Callback failure for ${toLoggableString()}", Platform.INFO, e)
          } else {
            //3
            responseCallback.onFailure(this@RealCall, e)
          }
        } catch (t: Throwable) {
          cancel()
          if (!signalledCallback) {
            val canceledException = IOException("canceled due to $t")
            canceledException.addSuppressed(t)
            //4
            responseCallback.onFailure(this@RealCall, canceledException)
          }
          throw t
        } finally {
          client.dispatcher.finished(this)
        }
      }
    }
  }

}
```

注释 1：`getResponseWithInterceptorChain()`方法直接返回了一个response对象

注释 2：把 response 通过 responseCallback 接口回调出去

注释 3：发生异常，回调失败接口

注释 4：发生异常，回调失败接口

> 到现在 Call 的异步请求全过程就看完了，而最关键的response是怎么得到的也就是 `getResponseWithInterceptorChain()`方法，会在下一篇文章分析。这篇只涉及到请求的流程

### 9.Call 的 execute 方法

Call 总共有两种请求方式一个 `enqueue`还有一个 `execute`方法，接下来看下同步执行的方法

我们上面可以得知 Call 其实是 RealCall，所以直接看 RealCall 的 execute 方法

```kotlin
//RealCall.kt
override fun execute(): Response {
  check(executed.compareAndSet(false, true)) { "Already Executed" }

  timeout.enter()
  callStart()
  try {
    client.dispatcher.executed(this)
    //1 拿到 response 直接返回
    return getResponseWithInterceptorChain()
  } finally {
    client.dispatcher.finished(this)
  }
}
```

> 这个 `execute`方法是运行在主线程的，直接通过 `getResponseWithInterceptorChain`方法获取 response 返回

### 10.总结

##### 10.1.OkHttp 是如何进行异步请求的？

* 通过 OkHttpClient 和 Request 参数构造出 RealCall 对象
* 通过 RealCall 对象的 `enqueue`方法进行网络请求
* 底层调用了 Dispatcher 的 `enqueue`方法，并把 AsyncCall 对象实例化，把 Callback 参数传入 AsyncCall 对象中
* 通过 `promoteAndExecute` 方法把集合中 AsyncCall 遍历出来，挨个执行 AsyncCall 的  `executeOn`方法，并把创建的线程池，作为参数传递进去
* 调用线程池的 `execute`方法，执行 AsyncCall 的 `run` 方法
* 在 AsyncCall 的 `run` 方法中，通过 `getResponseWithInterceptorChain`方法获取 response，并通过Callback的 `onResponse` 返回，如果出现异常则调用 `onFailure`方法。

##### 10.2.OkHttp 是如何进行同步请求的？

- 通过 OkHttpClient 和 Request 参数构造出 RealCall 对象
- 通过 RealCall 对象的 `execute`方法进行网络请求
- 通过 `getResponseWithInterceptorChain`方法获取 response，直接返回



### 11.源码地址

[OkHttpActivity.kt](https://github.com/jhbxyz/ArticleCode/blob/master/app/src/main/java/com/aboback/articlecode/okhttp/OkHttpActivity.kt)

### 12.原文地址

[OkHttp请求的流程的梳理](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/%E4%BC%98%E7%A7%80%E5%BC%80%E6%BA%90%E5%BA%93/2OkHttp%E7%9A%84%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%EF%BC%88%E4%B8%80%EF%BC%89.md)

### 13.参考资料

[OkHttp](https://square.github.io/okhttp/)













