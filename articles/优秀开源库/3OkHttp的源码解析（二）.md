# OkHttp

通过上一篇 [OkHttp请求的流程的梳理](https://juejin.im/post/6870021924337123336)，我们知道了 OkHttp 的整体请求流程，异步请求，同步请求，最后都是通过 `getResponseWithInterceptorChain`方法返回一个 Response 对象

这一篇就重点看一下 `getResponseWithInterceptorChain`方法，来看是如何获取一个 Response 的

> 整体就是通过一系列的拦截器，处理请求处理响应的过程

### 1.RealCall 的 getResponseWithInterceptorChain方法

```kotlin
//RealCall.kt
@Throws(IOException::class)
internal fun getResponseWithInterceptorChain(): Response {
  // Build a full stack of interceptors.
  val interceptors = mutableListOf<Interceptor>()
  //自己定义的拦截器
  interceptors += client.interceptors
  //重试和重定向拦截器
  interceptors += RetryAndFollowUpInterceptor(client)
  //桥接拦截器
  interceptors += BridgeInterceptor(client.cookieJar)
  //缓存拦截器
  interceptors += CacheInterceptor(client.cache)
  //网络拦截器
  interceptors += ConnectInterceptor
  if (!forWebSocket) {
    interceptors += client.networkInterceptors
  }
  //请求服务的拦截器
  interceptors += CallServerInterceptor(forWebSocket)

  //1
  val chain = RealInterceptorChain(
      call = this,
      interceptors = interceptors,
      index = 0,
      exchange = null,
      request = originalRequest,
      connectTimeoutMillis = client.connectTimeoutMillis,
      readTimeoutMillis = client.readTimeoutMillis,
      writeTimeoutMillis = client.writeTimeoutMillis
  )

  var calledNoMoreExchanges = false
  try {
    //2
    val response = chain.proceed(originalRequest)
    if (isCanceled()) {
      response.closeQuietly()
      throw IOException("Canceled")
    }
    //3
    return response
  } catch (e: IOException) {
    calledNoMoreExchanges = true
    throw noMoreExchanges(e) as Throwable
  } finally {
    if (!calledNoMoreExchanges) {
      noMoreExchanges(null)
    }
  }
}
```

> 先说下各个拦截器含义的作用
>
> * 自定义的拦截器：根据自己的需求，比如配置公共参数等，这个是在请求之前做的
> * RetryAndFollowUpInterceptor：重试和重定向拦截器，网络请求出错，或者服务器返会 301、302，OKHttp 会自动帮你重定向
> * BridgeInterceptor：桥接拦截器，拼接成一个标准的 Http 协议的请求，请求行，Header，Body 等
> * CacheInterceptor：缓存拦截器，根据 Header 来进行网络缓存的
> * ConnectInterceptor：连接拦截器，开启一个目标服务器的连接
> * 网络拦截器：在请求结果返回的时候，可以自定义一个，对接口数据进行处理，比如日志、log等
> * CallServerInterceptor：请求服务的拦截器，请求服务器
>
> 整个请求过程就是通过一个个拦截器来使请求完整，并在服务器返回时，经过一个个拦截器返一个我们需要的 Response

注释 1：一个具体化的拦截器的链，OkHttp 整个请求链的起点

> 构造 RealInterceptorChain时的参数
>
> * 把当前的 Call 对象传入，其实是 RealCall
> * 所有拦截器的集合传入
> * index = 0，当前拦截器集合的索引
> * 以及 originalRequest，最开始的 Request，没经过拦截器处理的 Request

注释 2：调用 `proceed` 方法，然后就可以根据 index 的值，分别调用不同的拦截器了

注释 3：把最终经过拦截器处理的 Response 返回

### 2.RealInterceptorChain的proceed方法

```kotlin
//RealInterceptorChain.kt
@Throws(IOException::class)
override fun proceed(request: Request): Response {
  check(index < interceptors.size)

  calls++

  // Call the next interceptor in the chain.
  //1
  val next = copy(index = index + 1, request = request)
  //2
  val interceptor = interceptors[index]

  //3
  val response = interceptor.intercept(next) ?: throw NullPointerException(
      "interceptor $interceptor returned null")

  return response
}
```

注释 1：调用 `copy` 函数，并把 index+1 获取一个新的 RealInterceptorChain对象，注意此时的 index 加 1 了，当前对象的 index 还是 0

> ```kotlin
> //RealInterceptorChain.kt
> internal fun copy(
>   index: Int = this.index,
>   exchange: Exchange? = this.exchange,
>   request: Request = this.request,
>   connectTimeoutMillis: Int = this.connectTimeoutMillis,
>   readTimeoutMillis: Int = this.readTimeoutMillis,
>   writeTimeoutMillis: Int = this.writeTimeoutMillis
> ) = RealInterceptorChain(call, interceptors, index, exchange, request, connectTimeoutMillis,
>     readTimeoutMillis, writeTimeoutMillis)
> ```

注释 2：因为此时的 `index = 0`我们在不考虑自定义拦截器的情况下，此时获取的就是 RetryAndFollowUpInterceptor：重试和重定向拦截器

注释 3：调用 RetryAndFollowUpInterceptor的 `intercept`方法

### 3.RetryAndFollowUpInterceptor重试和重定向拦截器

```kotlin
//RetryAndFollowUpInterceptor.kt
@Throws(IOException::class)
override fun intercept(chain: Interceptor.Chain): Response {
  val realChain = chain as RealInterceptorChain
  var request = chain.request
  val call = realChain.call
  var followUpCount = 0
  var priorResponse: Response? = null
  var newExchangeFinder = true
  var recoveredFailures = listOf<IOException>()
  //死循环，知道
  while (true) {
    call.enterNetworkInterceptorExchange(request, newExchangeFinder)

    var response: Response
    var closeActiveExchange = true
    try {
      if (call.isCanceled()) {
        throw IOException("Canceled")
      }

      try {
        //执行下一个 Interceptor 的方法
        response = realChain.proceed(request)
        newExchangeFinder = true
      } catch (e: RouteException) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
          throw e.firstConnectException.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e.firstConnectException
        }
        newExchangeFinder = false
        continue
      } catch (e: IOException) {
        // An attempt to communicate with a server failed. The request may have been sent.
        if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
          throw e.withSuppressed(recoveredFailures)
        } else {
          recoveredFailures += e
        }
        newExchangeFinder = false
        continue
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                .body(null)
                .build())
            .build()
      }

      val exchange = call.interceptorScopedExchange
      val followUp = followUpRequest(response, exchange)

      if (followUp == null) {
        if (exchange != null && exchange.isDuplex) {
          call.timeoutEarlyExit()
        }
        closeActiveExchange = false
        return response
      }

      val followUpBody = followUp.body
      if (followUpBody != null && followUpBody.isOneShot()) {
        closeActiveExchange = false
        return response
      }

      response.body?.closeQuietly()

      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw ProtocolException("Too many follow-up requests: $followUpCount")
      }

      request = followUp
      priorResponse = response
    } finally {
      call.exitNetworkInterceptorExchange(closeActiveExchange)
    }
  }
}
```







注释 4：

注释 5：





注释 1：

注释 2：

注释 3：

注释 4：

注释 5：

注释 1：

注释 2：

注释 3：

注释 4：

注释 5：

注释 1：

注释 2：

注释 3：

注释 4：

注释 5：