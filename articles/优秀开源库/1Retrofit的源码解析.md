# 一定能看懂的 Retrofit 最详细的源码解析！

**你在使用 Retrofit 的时候，是否会有如下几点疑惑？**

- 什么是动态代理？
- 整个请求的流程是怎样的？
- 底层是如何用 OkHttp 请求的？
- 方法上的注解是什么时候解析的，怎么解析的？
- Converter 的转换过程，怎么通过 Gson 转成对应的数据模型的？
- CallAdapter 的替换过程，怎么转成 RxJava 进行操作的？
- 如何支持 Kotlin 协程的 suspend 挂起函数的？
  
  > 关于 Kotlin 协程请求网络，首先写一个 Demo 来看一下用协程是怎么进行网络请求的，然后会再具体分析怎么转换成 Kotlin 的协程的请求

我会在文章中，通过源码，逐步解开疑惑，并且在最后文章结尾会再次总结，回答上面的几个问题。

**友情提示，本文略长但是没有废话，实打实的干货，学习需要耐心**

[Retrofit](https://square.github.io/retrofit/) 和 [OkHttp](https://square.github.io/okhttp/) 是目前最广泛使用的网络请求库了，所以有必要了解它的源码，学习它的优秀的代码与设计，来提升自己。

**本文的整体思路**

首先先看一下 Retrofit 的基本用法，根据示例代码，作为分析源码的依据，以及分析源码的入口，来一步一步看一下 Retrofit 的工作机制。

**本文的依赖**

```groovy
implementation 'com.squareup.okhttp3:okhttp:4.8.1'

implementation 'com.squareup.retrofit2:retrofit:2.9.0'
implementation 'com.squareup.retrofit2:converter-gson:2.5.0'
implementation 'com.squareup.retrofit2:adapter-rxjava2:2.7.2'

implementation 'com.google.code.gson:gson:2.8.6'

implementation 'io.reactivex.rxjava3:rxjava:3.0.0'
implementation 'io.reactivex.rxjava3:rxandroid:3.0.0'
```

### 1.什么是Retrofit

[Retrofit](https://github.com/square/retrofit)：A type-safe **HTTP client** for Android and Java。一个类型安全的 Http 请求的客户端。

底层的网络请求是基于 OkHttp 的，Retrofit 对其做了封装，提供了即方便又高效的网络访问框架。

### 2.Retrofit的基本用法

```kotlin
class RetrofitActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        //初始化一个Retrofit对象
        val retrofit = Retrofit.Builder()
            .baseUrl("https://api.github.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
        //创建出GitHubApiService对象
        val service = retrofit.create(GitHubApiService::class.java)
        //返回一个 Call 对象
        val repos = service.listRepos("octocat")
        //调用 enqueue 方法在回调方法里处理结果
        repos.enqueue(object : Callback<List<Repo>?> {
            override fun onFailure(call: Call<List<Repo>?>, t: Throwable) {
								t.printStackTrace()
            }

            override fun onResponse(call: Call<List<Repo>?>, response: Response<List<Repo>?>) {
                "response.code() = ${response.code()}".logE()
            }
        })

    }
}
```

```kotlin
//自己定义的 API 请求接口
interface GitHubApiService {
    @GET("users/{user}/repos")
    fun listRepos(@Path("user") user: String?): Call<List<Repo>>
}
```

以上就是 Retrofit 的基本用法。

没什么好讲的，写这个例子就是为了分析源码做准备，有一个源码分析的入口。

### 3.源码分析的准备工作

先看几个表面上的类

* Retrofit：总揽全局一个类，一些配置，需要通过其内部 **Builder** 类构建，比如 CallAdapter、Converter 等
* GitHubApiService：自己写的 API 接口，通过 Retrofit 的 `create` 方法进行实例化
* Call：Retrofit 的 Call，是执行网络请求的是一个顶层接口，需要看源码，具体实现类实际是一个 **OkHttpCall**，下面会具体说
* Callback：请求结果回调

接下来重点来了，进行源码分析。

### 4.源码分析

分析的入口是我们代码例子中的`repos.enqueue(object : Callback<List<Repo>?> {…})`方法

点进去，看到是 Call 的`enqueue`方法

```java
public interface Call<T> extends Cloneable {

  void enqueue(Callback<T> callback);

}
```

这是一个接口，是我们 GitHubApiService 接口中定义的 `listRepos` 方法中返回的 Call 对象，现在就要看GitHubApiService 的初始化，以及具体返回的是 **Call** 对象是谁。

然后重点就要看 `retrofit.create(GitHubApiService::class.java)` 方法，来看下 GitHubApiService 具体是怎么创建的，以及 Call 对象的实现类

### 5.Retrofit 的 create 方法

```java
//Retrofit.java
public <T> T create(final Class<T> service) {
  //1
  validateServiceInterface(service);
  //2
  return (T)
      Proxy.newProxyInstance(
          service.getClassLoader(),
          new Class<?>[] {service},
          new InvocationHandler() {
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            @Override
            public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                throws Throwable {
              // If the method is a method from Object then defer to normal invocation.
              if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
              }
              args = args != null ? args : emptyArgs;
              return platform.isDefaultMethod(method)
                  ? platform.invokeDefaultMethod(method, service, proxy, args)
                  : loadServiceMethod(method).invoke(args);
            }
          });
}
```

注释 1：这个方法，就是验证我们定义的 GitHubApiService 是一个接口，且不是泛型接口，并且会判断是否进行方法的**提前验证**，为了更好的把错误暴露的编译期，这个不是我们的重点内容，具体代码就不看了。

注释 2 ：是一个动态代理的方法，来返回 GitHubApiService 的实例

> 动态代理？嗯？什么是动态代理，接下来，我就写一个具体的例子来展示一个动态代理的具体用法，以及什么是动态代理？
>
> 先插播一段动态代理代码，这个是理解 Retrofit 的工作机制所必须的。

### 6.动态代理的示例

#### 6.1.写一个动态代理的 Demo

下面是一个 **Java** 项目，模拟一个 Retrofit 的请求过程

```java
//模拟 Retrofit,定义 API 请求接口
public interface GitHubApiService {
    void listRepos(String user);
}
```
```java
public class ProxyDemo {
  	//程序的入口方法
    public static void main(String[] args) {
        //通过动态代理获取 ApiService 的对象
        GitHubApiService apiService = (GitHubApiService) Proxy.newProxyInstance(
                GitHubApiService.class.getClassLoader(),
                new Class[]{GitHubApiService.class},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

                        System.out.println("method = " + method.getName() + "   args = " + Arrays.toString(args));

                        return null;
                    }
                });

        System.out.println(apiService.getClass());
        //调用 listRepos 方法
        apiService.listRepos("octcat");
    }

}
```

执行 `main` 方法

当我们调用 `apiService.listRepos("octcat");`方法时，打印出来如下结果

```log
class com.sun.proxy.$Proxy0
method = listRepos   args = [octcat]
```

可以看到当我们调用`listRepos`方法的时候，InvocationHandler 的 `invoke`方法中拦截到了我们的方法，参数等信息。**Retrofit 的原理其实就是这样，拦截到方法、参数，再根据我们在方法上的注解，去拼接为一个正常的OkHttp 请求，然后执行。**

日志的第一行，在运行时这个类一个`$Proxy0`的类。实际上，在运行期 GitHubApiService 的接口会动态的创建出**实现类**也就是这个 `$Proxy0`类，它大概长下面这个样子，具体的看[鸿洋](https://blog.csdn.net/lmj623565791)这篇文章  [从一道面试题开始说起 枚举、动态代理的原理](https://blog.csdn.net/lmj623565791/article/details/79278864)

我做了一个点改动，方便查看，本质上都是一样的

```java
class $Proxy0 extends Proxy implements GitHubApiService {

    protected $Proxy0(InvocationHandler h) {
        super(h);
    }

    @Override
    public void listRepos(String user) {

        Method method = Class.forName("GitHubApiService").getMethod("listRepos", String.class);

        super.h.invoke(this, method, new Object[]{user});
    }
}
```

我们在调用`listRepos`方法的时候，实际上调用的是 InvocationHandler 的 `invoke` 方法。

#### 6.2总结

* 在 ProxyDemo 代码运行中，会动态创建 GitHubApiService 接口的实现类，作为代理对象，执行InvocationHandler 的 `invoke` 方法。
* 动态指的是在运行期，而代理指的是实现了GitHubApiService 接口的具体类，实现了接口的方法，称之为代理
* 本质上是在运行期，生成了 GitHubApiService 接口的实现类，调用了 InvocationHandler 的 `invoke`方法。

现在解决了第一个疑问：**什么是动态代理**

好的，动态代理已经知道是啥了，回到我们 `retrofit.create(GitHubApiService::class.java)`方法

### 7.再看 Retrofit 的 create 方法

```java
//Retrofit.java
public <T> T create(final Class<T> service) {
  validateServiceInterface(service);
  return (T)
      Proxy.newProxyInstance(
          service.getClassLoader(),//1
          new Class<?>[] {service},//2
          new InvocationHandler() {//3
            private final Platform platform = Platform.get();
            private final Object[] emptyArgs = new Object[0];

            @Override
            public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                throws Throwable {
              // If the method is a method from Object then defer to normal invocation.
              //4
              if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
              }
              args = args != null ? args : emptyArgs;
              //5
              return platform.isDefaultMethod(method)
                  ? platform.invokeDefaultMethod(method, service, proxy, args)
                  : loadServiceMethod(method).invoke(args);
            }
          });
}
```

注释 1：获取一个 ClassLoader 对象

注释 2：GitHubApiService 的字节码对象传到数组中去，也即是我们要代理的具体接口。

注释 3：InvocationHandler 的 `invoke` 是关键，从上面动态代理的 Demo 中，我们知道，在GitHubApiService声明的 `listRepos`方法在调用时，会执行 InvocationHandler 的`invoke`的方法体。

注释 4：因为有代理类的生成，默认继承 Object 类，所以如果是 Object.class 走，默认调用它的方法

注释 5：如果是默认方法（比如 Java8 ），就执行 platform 的默认方法。**否则执行`loadServiceMethod`方法的`invoke`方法**

`loadServiceMethod(method).invoke(args);`这个方法是我们这个 Retrofit **最关键的代码**，也是分析的重点入口

#### 7.1.先看loadServiceMethod方法

我们先看`loadServiceMethod`方法返回的是什么对象，然后再看这个对象的 `invoke` 方法

```java
//Retrofit.java
private final Map<Method, ServiceMethod<?>> serviceMethodCache = new ConcurrentHashMap<>();

ServiceMethod<?> loadServiceMethod(Method method) {
  //1
  ServiceMethod<?> result = serviceMethodCache.get(method);
  if (result != null) return result;

  synchronized (serviceMethodCache) {
    result = serviceMethodCache.get(method);
    if (result == null) {
      //2
      result = ServiceMethod.parseAnnotations(this, method);
      //3
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```

注释 1：从 ConcurrentHashMap 中取一个 ServiceMethod 如果存在直接返回

注释 2：通过 `ServiceMethod.parseAnnotations(this, method);`方法创建一个 ServiceMethod 对象

注释 3：用 Map 把创建的 ServiceMethod 对象缓存起来，因为我们的请求方法可能会调用多次，缓存提升性能。

看一下 `ServiceMethod.parseAnnotations(this, method);`方法具体返回的对象是什么，然后再看它的 `invoke` 方法

#### 7.2.ServiceMethod的parseAnnotations方法

这个方法接下来还会看，这里我们只看现在需要的部分。

```java
//ServiceMethod.java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
  ...
  //1
  return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}
```

返回的是一个**HttpServiceMethod**对象，那么接下来看下它的 invoke 方法

#### 7.3.HttpServiceMethod 的 invoke 方法

```java
//HttpServiceMethod.java
@Override
final @Nullable ReturnT invoke(Object[] args) {
  //1
  Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
  //2
  return adapt(call, args);
}
```

注释 1：创建了一个Call对象，是 OkHttpCall，这个不就是在 GitHubApiService 这个接口声明的 Call 对象吗？

然后再看 OkHttpCall 的`enqueue`方法，不就知道是怎么进行请求，怎么回调的了吗？

注释 2：是一个 adapt 方法，在不使用 Kotlin 协程的情况下，其实调用的是子类 CallAdapted 的 `adapt`，这个会在下面具体分析，包括 Kotlin 协程的 suspend 函数

现在我们已经知道了 GitHubApiService 接口中定义的 `listRepos`中的 Call 对象，是 OkHttpCall，接下里看OkHttpCall 的 `enqueue` 方法

### 8.OkHttpCall的enqueue方法

这段代码比较长，但这个就是这个请求的关键，以及怎么使用 OkHttp 进行请求的，如果解析 Response 的，如何回调的。

```java
//OkHttpCall.java
@Override
public void enqueue(final Callback<T> callback) {
  Objects.requireNonNull(callback, "callback == null");

  //1
  okhttp3.Call call;
  Throwable failure;

  synchronized (this) {
    if (executed) throw new IllegalStateException("Already executed.");
    executed = true;

    call = rawCall;
    failure = creationFailure;
    if (call == null && failure == null) {
      try {
        //2
        call = rawCall = createRawCall();
      } catch (Throwable t) {
        throwIfFatal(t);
        failure = creationFailure = t;
      }
    }
  }

  if (failure != null) {
    callback.onFailure(this, failure);
    return;
  }

  if (canceled) {
    call.cancel();
  }

  //3
  call.enqueue(
      new okhttp3.Callback() {
        @Override
        public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
          Response<T> response;
          try {
            //4
            response = parseResponse(rawResponse);
          } catch (Throwable e) {
            throwIfFatal(e);
            callFailure(e);
            return;
          }

          try {
            //5
            callback.onResponse(OkHttpCall.this, response);
          } catch (Throwable t) {
            throwIfFatal(t);
            t.printStackTrace(); // TODO this is not great
          }
        }

        @Override
        public void onFailure(okhttp3.Call call, IOException e) {
          callFailure(e);
        }

        private void callFailure(Throwable e) {
          try {
            //6
            callback.onFailure(OkHttpCall.this, e);
          } catch (Throwable t) {
            throwIfFatal(t);
            t.printStackTrace(); // TODO this is not great
          }
        }
      });
}
```

注释 1：声明一个 okhttp3.Call 对象，用来进行网络请求

注释 2：给 okhttp3.Call 对象进行赋值，下面会具体看代码，如何创建了一个 okhttp3.Call 对象

注释 3：调用 okhttp3.Call 的 `enqueue` 方法，进行真正的网络请求

注释 4：解析响应，下面会具体看代码

注释 5：成功的回调

注释 6：失败的回调

到现在，我们文章开头两个疑问得到解释了

> 整个请求的流程是怎样的？
>
> 底层是如何用 OkHttp 请求的？

我们还要看下一个 okhttp3.Call 对象是怎么创建的，我们写的注解参数是怎么解析的，响应结果是如何解析的，也就是我们在 Retrofit 中配置 `addConverterFactory(GsonConverterFactory.create())`是如何直接拿到数据模型的。

#### 8.1.okhttp3.Call 对象是怎么创建的

看下 `call = rawCall = createRawCall();`方法

```java
//OkHttpCall.java
private final okhttp3.Call.Factory callFactory;

private okhttp3.Call createRawCall() throws IOException {
  //1 callFactory是什么
  okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}
```

通过 callFactory 创建的（callFactory应该是 OkHttpClient），看一下 callFactory 的赋值过程

```java
//OkHttpCall.java
OkHttpCall(
    RequestFactory requestFactory,
    Object[] args,
    okhttp3.Call.Factory callFactory,
    Converter<ResponseBody, T> responseConverter) {
  this.requestFactory = requestFactory;
  this.args = args;
  //通过 OkHttpCall 构造直接赋值
  this.callFactory = callFactory;
  this.responseConverter = responseConverter;
}
```

在 OkHttpCall 构造中直接赋值，那接下来就继续看 OkHttpCall 的初始化过程

```java
//HttpServiceMethod.java
private final okhttp3.Call.Factory callFactory;

@Override
final @Nullable ReturnT invoke(Object[] args) {
  //在 OkHttpCall 实例化时赋值， callFactory 是 HttpServiceMethod 的成员变量
  Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
  return adapt(call, args);
}

//callFactory 是在 HttpServiceMethod 的构造中赋值的
HttpServiceMethod(
    RequestFactory requestFactory,
    okhttp3.Call.Factory callFactory,
    Converter<ResponseBody, ResponseT> responseConverter) {
  this.requestFactory = requestFactory;
	//通过 HttpServiceMethod 构造直接赋值
  this.callFactory = callFactory;
  this.responseConverter = responseConverter;
}
```

发现 callFactory 的值是在创建 HttpServiceMethod 时赋值的，继续跟！

在 7.2 小节，有一行代码`HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);`我们没有跟进去，现在看一下 **HttpServiceMethod** 是怎么创建的

```java
//HttpServiceMethod.java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
  boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
  boolean continuationWantsResponse = false;
  boolean continuationBodyNullable = false;

	//1
  okhttp3.Call.Factory callFactory = retrofit.callFactory;
  if (!isKotlinSuspendFunction) {
    //2
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
  } else if (continuationWantsResponse) {
    //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
    return (HttpServiceMethod<ResponseT, ReturnT>)
        new SuspendForResponse<>(
            requestFactory,
            callFactory,
            responseConverter,
            (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
  } else {
    //noinspection unchecked Kotlin compiler guarantees ReturnT to be Object.
    return (HttpServiceMethod<ResponseT, ReturnT>)
        new SuspendForBody<>(
            requestFactory,
            callFactory,
            responseConverter,
            (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
            continuationBodyNullable);
  }
}
```

注释 1：callFactory 的值是从 Retrofit 这个对象拿到的

注释 2：如果不是 Kotlin 的挂起函数，返回是的 CallAdapted 对象

```java
static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {}
```

> CallAdapted 是 HttpServiceMethod 的子类，会调用 `adapt`方法进行 CallAdapter 的转换，我们后面会详细看。

继续看 Retrofit 的 callFactory 的值Retrofit是通过Builder构建的，看下Builder类

```java
//Retrofit.java
public static final class Builder {
    public Retrofit build() {

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        //1
        callFactory = new OkHttpClient();
      }

      return new Retrofit(
          callFactory,
          baseUrl,
          unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories),
          callbackExecutor,
          validateEagerly);
    }

}
```

原来 callFactory 实际是一个 OkHttpClient 对象，也就是 OkHttpClient 创建了一个 Call 对象，嗯就是 OKHttp 网络请求的那一套。

在创建okhttp3.Call 对象的 `callFactory.newCall(requestFactory.create(args));`方法中的 `requestFactory.create(args)`方法会返回一个 Request 的对象，这个我们也会在下面看是如何构造一个 OkHttp 的 Request 请求对象的。

#### 8.2.请求注解参数是怎么解析的

看 `ServiceMethod.parseAnnotations(this, method);`方法

```java
//ServiceMethod.java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
  //1
  RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
	...
  //2
  return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}
```

注释 1：通过 RequestFactory 解析注解，然后返回 RequestFactory 对象

注释 2：把 RequestFactory 对象往 HttpServiceMethod 里面传递，下面会具体看 RequestFactory 对象具体干什么用了？

继续跟代码`RequestFactory.parseAnnotations`

```java
//RequestFactory.java
final class RequestFactory {
  static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    //看build方法
    return new Builder(retrofit, method).build();
  }
  
  //build方法
  RequestFactory build() {
    //1
    for (Annotation annotation : methodAnnotations) {
      parseMethodAnnotation(annotation);
    }

   ....

    return new RequestFactory(this);
  }
}
```

遍历 GitHubApiService 这个 API 接口上定义的方法注解，然后解析注解

继续跟代码`parseMethodAnnotation`

```java
//RequestFactory.java
private void parseMethodAnnotation(Annotation annotation) {
  if (annotation instanceof DELETE) {
    parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
  } else if (annotation instanceof GET) {
    parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
  }
  ...
  else if (annotation instanceof POST) {
    parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
  }
 	....
  else if (annotation instanceof Multipart) {
    if (isFormEncoded) {
      throw methodError(method, "Only one encoding annotation is allowed.");
    }
    isMultipart = true;
  } else if (annotation instanceof FormUrlEncoded) {
    if (isMultipart) {
      throw methodError(method, "Only one encoding annotation is allowed.");
    }
    isFormEncoded = true;
  }
}
```

就是解析方法上的注解，来存到 RequestFactory 的内部。

其实 RequestFactory 这个类还有 `parseParameter` 和 `parseParameterAnnotation`这个就是解析方法参数声明上的具体参数的注解，会在后面分析 Kotlin suspend 挂起函数具体讲。

**总之：**具体代码就是分析方法上注解上面的值，方法参数上，这个就是细节问题了

**总结就是**：分析方法上的各个注解，方法参数上的注解，最后返回 RequestFactory 对象，给下面使用。

**Retrofit 的大框架简单，细节比较复杂。**

RequestFactory 对象返回出去，具体干嘛用了？大胆猜一下，解析出注解存到 RequestFactory 对象，这个对象身上可有各种请求的参数，然后肯定是类创建 OkHttp 的 Request请求对象啊，因为是用 OkHttp 请求的，它需要一个 Request 请求对象

#### 8.3.RequestFactory 对象返回出去，具体干嘛用了?

下面我就用一个代码块贴了，看着更直接，我会具体表明属于哪个类的

```java
//ServiceMethod.java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
  //解析注解参数，获取 RequestFactory 对象
  RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
  //把 RequestFactory 对象传给 HttpServiceMethod
  return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
}

//注意换类了
//HttpServiceMethod.java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
 
 	...

  okhttp3.Call.Factory callFactory = retrofit.callFactory;
  //不是 Kotlin 的挂起函数
  if (!isKotlinSuspendFunction) {
    //把requestFactory传给 CallAdapted
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
  } 
  ....
}

//HttpServiceMethod.java
//CallAdapted 是 HttpServiceMethod 的内部类也是 HttpServiceMethod 的子类
CallAdapted(
    RequestFactory requestFactory,
    okhttp3.Call.Factory callFactory,
    Converter<ResponseBody, ResponseT> responseConverter,
    CallAdapter<ResponseT, ReturnT> callAdapter) {
  //这里把 requestFactory 传给 super 父类的构造参数里了，也就是 HttpServiceMethod
  super(requestFactory, callFactory, responseConverter);
  this.callAdapter = callAdapter;
}

//HttpServiceMethod.java
HttpServiceMethod(
    RequestFactory requestFactory,
    okhttp3.Call.Factory callFactory,
    Converter<ResponseBody, ResponseT> responseConverter) {
  // HttpServiceMethod 的 requestFactory 成员变量保存这个 RequestFactory 对象
  this.requestFactory = requestFactory;
  this.callFactory = callFactory;
  this.responseConverter = responseConverter;
}

//因为会调用  HttpServiceMethod 的 invoke 方法
//会把这个 RequestFactory 对象会继续传递给 OkHttpCall 类中
//注意换类了
//OkHttpCall.java
OkHttpCall(
    RequestFactory requestFactory,
    Object[] args,
    okhttp3.Call.Factory callFactory,
    Converter<ResponseBody, T> responseConverter) {
  //给 OkHttpCall 的requestFactory成员变量赋值
  this.requestFactory = requestFactory;
  this.args = args;
  this.callFactory = callFactory;
  this.responseConverter = responseConverter;
}


```

经过层层传递 RequestFactory 这个实例终于是到了 HttpServiceMethod 类中，最终传到了 OkHttpCall 中，那这个 RequestFactory 对象在什么时候使用呢？ RequestFactory 会继续在OkHttpCall中传递，因为 **OkHttpCall** 才是进行请求的。

在OkHttpCall的 创建 Call 对象时

```java
//OkHttpCall.java
private okhttp3.Call createRawCall() throws IOException {
  //1
  okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}
```

注释 1：调用了`requestFactory.create(args)`

注意：此时的RequestFactory的各个成员变量在解析注解那一步都赋值了

```java
//RequestFactory.java
okhttp3.Request create(Object[] args) throws IOException {
  ...
  RequestBuilder requestBuilder =
      new RequestBuilder(
          httpMethod,
          baseUrl,
          relativeUrl,
          headers,
          contentType,
          hasBody,
          isFormEncoded,
          isMultipart);
  ...
  return requestBuilder.get().tag(Invocation.class, new Invocation(method, argumentList)).build();
}
```

**最终 requestFactory 的值用来构造 okhttp3.Request 的对象**

以上就是解析注解，构造出okhttp3.Request的对象全过程了。

也就解答了**方法上的注解是什么时候解析的，怎么解析的？**这个问题

#### 8.4.请求响应结果是如何解析的

比如我们在构造 Retrofit 的时候加上 `addConverterFactory(GsonConverterFactory.create())`这行代码，我们的响应结果是如何通过 Gson 直接解析成数据模型的？

在 OkHttpCall 的`enqueue`方法中

```java
//OkHttpCall.java
@Override
public void enqueue(final Callback<T> callback) {

  okhttp3.Call call;
 	...
  call.enqueue(
      new okhttp3.Callback() {
        @Override
        public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse) {
          Response<T> response;
          try {
            //1 解析响应
            response = parseResponse(rawResponse);
          } catch (Throwable e) {
            throwIfFatal(e);
            callFailure(e);
            return;
          }
        }
	...
      });
}
```

注释 1：通过`parseResponse`解析响应返回给回调接口

```java
//OkHttpCall.java
private final Converter<ResponseBody, T> responseConverter;

Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();

 	...

  ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
  try {
    //1 通过 responseConverter 转换 ResponseBody
    T body = responseConverter.convert(catchingBody);
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    // If the underlying source threw an exception, propagate that rather than indicating it was
    // a runtime exception.
    catchingBody.throwIfCaught();
    throw e;
  }
}
```

注释 1：通过 responseConverter 调用`convert`方法

首先那看 responseConverter 是什么以及赋值的过程，然后再看`convert`方法

```java
//OkHttpCall.java
private final Converter<ResponseBody, T> responseConverter;

OkHttpCall(
    RequestFactory requestFactory,
    Object[] args,
    okhttp3.Call.Factory callFactory,
    Converter<ResponseBody, T> responseConverter) {
  this.requestFactory = requestFactory;
  this.args = args;
  this.callFactory = callFactory;
  //在构造中赋值
  this.responseConverter = responseConverter;
}

// OkHttpCall 在 HttpServiceMethod 类中实例化
//注意换类了
//HttpServiceMethod.java
private final Converter<ResponseBody, ResponseT> responseConverter;

HttpServiceMethod(
    RequestFactory requestFactory,
    okhttp3.Call.Factory callFactory,
    Converter<ResponseBody, ResponseT> responseConverter) {
  this.requestFactory = requestFactory;
  this.callFactory = callFactory;
   //在构造中赋值
  this.responseConverter = responseConverter;
}

//HttpServiceMethod 在子类 CallAdapted 调用 super方法赋值
CallAdapted(
    RequestFactory requestFactory,
    okhttp3.Call.Factory callFactory,
    Converter<ResponseBody, ResponseT> responseConverter,
    CallAdapter<ResponseT, ReturnT> callAdapter) {
  //在CallAdapted中调用super赋值
  super(requestFactory, callFactory, responseConverter);
  this.callAdapter = callAdapter;
}


```

继续看 CallAdapted 的初始化中 **responseConverter** 的赋值过程

```java
//HttpServiceMethod.java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
   ...
  CallAdapter<ResponseT, ReturnT> callAdapter =
      createCallAdapter(retrofit, method, adapterType, annotations);
 
  //1 实例化responseConverter
  Converter<ResponseBody, ResponseT> responseConverter =
      createResponseConverter(retrofit, method, responseType);

  okhttp3.Call.Factory callFactory = retrofit.callFactory;
  if (!isKotlinSuspendFunction) {
    //2 CallAdapted的实例化赋值
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
  } 
  ...
}
```

继续跟代码 `createResponseConverter`方法

```java
//HttpServiceMethod.java
private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
    Retrofit retrofit, Method method, Type responseType) {
  Annotation[] annotations = method.getAnnotations();
  try {
    //调用的是 retrofit的方法
    return retrofit.responseBodyConverter(responseType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(method, e, "Unable to create converter for %s", responseType);
  }
}
//注意换类了
//Retrofit.java
final List<Converter.Factory> converterFactories;

public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
  //继续跟 nextResponseBodyConverter
  return nextResponseBodyConverter(null, type, annotations);
}

public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
    @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
	...
  //1 从 converterFactories工厂中遍历取出
  int start = converterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = converterFactories.size(); i < count; i++) {
    Converter<ResponseBody, ?> converter =
        converterFactories.get(i).responseBodyConverter(type, annotations, this);
    if (converter != null) {
      //noinspection unchecked
      return (Converter<ResponseBody, T>) converter;
    }
  }
  ...
}

```

注释 1：从 converterFactories 遍历取出一个来调用 `responseBodyConverter` 方法，注意根据 responseType **返回值类型**来取到对应的 Converter，如果不为空，直接返回此 Converter 对象

看一下 converterFactories 这个对象的赋值过程

```java
//Retrofit.java
final List<Converter.Factory> converterFactories;

Retrofit(
    okhttp3.Call.Factory callFactory,
    HttpUrl baseUrl,
    List<Converter.Factory> converterFactories,
    List<CallAdapter.Factory> callAdapterFactories,
    @Nullable Executor callbackExecutor,
    boolean validateEagerly) {
  this.callFactory = callFactory;
  this.baseUrl = baseUrl;
  this.converterFactories = converterFactories; // Copy+unmodifiable at call site.
  //通过 Retrofit 的构造赋值，Retrofit的 初始化是通过内部 Builder 类的build方法
  this.callAdapterFactories = callAdapterFactories; // Copy+unmodifiable at call site.
  this.callbackExecutor = callbackExecutor;
  this.validateEagerly = validateEagerly;
}

//Retrofit.java 内部类 Builder 类的build方法
//Builder.java
 public Retrofit build() {
   
   ...
      // Make a defensive copy of the converters.
     //1
      List<Converter.Factory> converterFactories =
          new ArrayList<>(
              1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());
   		//2
      converterFactories.add(new BuiltInConverters());
   		//3
      converterFactories.addAll(this.converterFactories);
   		//4
      converterFactories.addAll(platform.defaultConverterFactories());

      return new Retrofit(
          callFactory,
          baseUrl,
          unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories),
          callbackExecutor,
          validateEagerly);
    }
```

注释 1：初始化 converterFactories 这个 list

注释 2：添加默认的构建的转换器，其实是 StreamingResponseBodyConverter 和 BufferingResponseBodyConverter

注释 3：就是自己添加的转换配置 `addConverterFactory(GsonConverterFactory.create())`

```java
//Retrofit.java 内部类 Builder.java
public Builder addConverterFactory(Converter.Factory factory) {
  converterFactories.add(Objects.requireNonNull(factory, "factory == null"));
  return this;
}
```

注释 4：如果是 Java8 就是一个 OptionalConverterFactory 的转换器否则就是一个空的

注意：是怎么找到GsonConverterFactory来调用 Gson 的 convert方法的呢？在遍历converterFactories时会根据 **responseType**来找到对应的转换器。

具体 GsonConverterFactory 的 `convert` 方法就是 Gson 的逻辑了，就不是 Retrofit 的重点了。

到现在**Converter 的转换过程**，我们也就清楚了。

**还有一个问题，我们写的 API 接口是如何支持 RxJava 的**

### 9.CallAdapter的替换过程

#### 9.1.使用 RxJava 进行网络请求

怎么转成 RxJava

比如：我们在初始化一个Retrofit时加入 `addCallAdapterFactory(RxJava2CallAdapterFactory.create())`这行

```java
//初始化一个Retrofit对象
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.github.com/")
  	//加入 RxJava2CallAdapterFactory 支持
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

加入 RxJava2 的配置支持后，把 RxJava2CallAdapterFactory 存到 callAdapterFactories 这个集合中，记住这一点，下面要用到。

```kotlin
interface GitHubApiService {
    @GET("users/{user}/repos")
    fun listReposRx(@Path("user") user: String?): Single<Repo>
}
```

我们就可以这么请求接口了

```java
//创建出GitHubApiService对象
val service = retrofit.create(GitHubApiService::class.java)
service.listReposRx("octocat")
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe({ repo ->
        "response name = ${repo[0].name}".logE()
    }, { error ->
        error.printStackTrace()
    })
```

我们可以在自己定义的 API 接口中直接返回一个 RxJava 的 Single 对象的，来进行操作了。

我们下面就来看下是如何把请求对象转换成一个 Single 对象的

```java
//Retrofit.java 内部类 Builder.java
public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
  callAdapterFactories.add(Objects.requireNonNull(factory, "factory == null"));
  return this;
}
```

把 RxJava2CallAdapterFactory 存到了callAdapterFactories 这个 list 中了。

接下来我们看下是如何使用 callAdapterFactories 的 RxJava2CallAdapterFactory 中的这个 CallAdapter 的吧

这就要看我们之前看到了一个类了 HttpServiceMethod 的`parseAnnotations`之前看过它的代码，只是上次看的是**Converter是如何赋值的**也就是第 8.4 小节，这次看 CallAdapter 是如何被赋值使用的。

#### 9.2CallAdapter是如何被赋值过程

**HttpServiceMethod的`parseAnnotations`方法**

```java
//HttpServiceMethod.java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
  
  ....
  //1
  CallAdapter<ResponseT, ReturnT> callAdapter =
      createCallAdapter(retrofit, method, adapterType, annotations);

  okhttp3.Call.Factory callFactory = retrofit.callFactory;
  if (!isKotlinSuspendFunction) {
    //2
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
  } 
 	...
}
```

注释 1：初始化 CallAdapter

注释 2：给 **CallAdapted** 中的 callAdapter 变量赋值，然后调用它的`adapt` 方法。

我们先找到具体 CallAdapter 赋值的对象，然后看它的`adapt`就知道了，是如何转换的了

接下来就是跟代码的过程了

```java
//HttpServiceMethod.java
private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(
    Retrofit retrofit, Method method, Type returnType, Annotation[] annotations) {
  try {
    //noinspection unchecked
   	//调用retrofit的callAdapter方法
    return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(method, e, "Unable to create call adapter for %s", returnType);
  }
}

//Retrofit.java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
  //调用nextCallAdapter
  return nextCallAdapter(null, returnType, annotations);
}

public CallAdapter<?, ?> nextCallAdapter(
    @Nullable CallAdapter.Factory skipPast, Type returnType, Annotation[] annotations) {
 
  ...

  //遍历 callAdapterFactories
  int start = callAdapterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
    //是具体CallAdapterFactory的 get 方法
    CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
    if (adapter != null) {
      return adapter;
    }
  }
  ...
}

```

遍历 callAdapterFactories 根据 **returnType类型** 来找到对应的 CallAdapter 返回

比如：我们在 GitHubApiService 的 returnType 类型为 Single，那么返回的就是 RxJava2CallAdapterFactory 所获取的 CallAdapter

```java
interface GitHubApiService {
    @GET("users/{user}/repos")
    fun listReposRx(@Path("user") user: String?): Single<Repo>
}
```

RxJava2CallAdapterFactory的 `get`方法

```java
//RxJava2CallAdapterFactory.java
@Override public @Nullable CallAdapter<?, ?> get(
    Type returnType, Annotation[] annotations, Retrofit retrofit) {
  Class<?> rawType = getRawType(returnType);

  if (rawType == Completable.class) {
    return new RxJava2CallAdapter(Void.class, scheduler, isAsync, false, true, false, false,
        false, true);
  }

  boolean isFlowable = rawType == Flowable.class;
  //当前是Single类型
  boolean isSingle = rawType == Single.class;
  boolean isMaybe = rawType == Maybe.class;
  if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
    return null;
  }
  ...
	//返回 RxJava2CallAdapter对象，isSingle参数为 true
  return new RxJava2CallAdapter(responseType, scheduler, isAsync, isResult, isBody, isFlowable,
      isSingle, isMaybe, false);
}
```

返回的是 RxJava2CallAdapter 对象，并且根据 rawType 判断当前是个什么类型

看下 RxJava2CallAdapter 的`adapt`方法

```java
//RxJava2CallAdapter.java
@Override public Object adapt(Call<R> call) {
  //1 把Call包装成一个Observable对象
  Observable<Response<R>> responseObservable = isAsync
      ? new CallEnqueueObservable<>(call)
      : new CallExecuteObservable<>(call);

  Observable<?> observable;
  if (isResult) {
    observable = new ResultObservable<>(responseObservable);
  } else if (isBody) {
    observable = new BodyObservable<>(responseObservable);
  } else {
    observable = responseObservable;
  }

  if (scheduler != null) {
    observable = observable.subscribeOn(scheduler);
  }

  if (isFlowable) {
    return observable.toFlowable(BackpressureStrategy.LATEST);
  }
  //2
  if (isSingle) {
    return observable.singleOrError();
  }
  if (isMaybe) {
    return observable.singleElement();
  }
  if (isCompletable) {
    return observable.ignoreElements();
  }
  return RxJavaPlugins.onAssembly(observable);
}
```

注释 1：把 Call 包装成一个 Observable 对象

注释2：如果是 Single 则调用`observable.singleOrError();`方法

到目前为止，**CallAdapter 怎么变成一个 RxJava2CallAdapter 以及它的具体调用**，我们也就清楚了。

### 10.Retrofit 如何支持 Kotlin 协程的 suspend 挂起函数的？

整个流程中还有一点我们没有分析 Retrofit **如何支持 Kotlin 协程的 suspend 挂起函数的？**

首先写一个 Demo 来看一下协程是怎么进行网络请求的

#### 10.1.Kotlin 协程请求网络的 Demo

**添加依赖**

```groovy
def kotlin_coroutines = '1.3.7'
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:$kotlin_coroutines"
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:$kotlin_coroutines"

implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.2.0'
implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.2.0'
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.2.0'
```

**定义请求接口，写一个挂起函数**

```kotlin
interface GitHubApiService {
    //使用 Kotlin 协程 ，定义一个挂起函数
    @GET("users/{user}/repos")
    suspend fun listReposKt(@Path("user") user: String?): List<Repo>
}
```

**请求接口**

```kotlin
//创建出GitHubApiService对象
val service = retrofit.create(GitHubApiService::class.java)
//lifecycle提供的协程的Scope，因为 suspend 函数需要运行在协程里面
lifecycleScope.launchWhenResumed {
    try {
        val repo = service.listReposKt("octocat")
        "response name = ${repo[0].name}".logE()
    } catch (e: Exception) {
        e.printStackTrace()
      	//出错逻辑
      	//ignore
    }
}
```

以上就是一个，用 Kotlin 协程进行网络请求的，Retrofit 是支持 Kotlin 协程的，接下来看下，Retrofit 是怎么支持的。

#### 10.2.分析Kotlin 协程的挂起函数的准备工作

首先在开始之前，我们得先得从代码角度知道，Kotlin 的 suspend 函数对应的 Java 类是什么样子，不然，就一个 **suspend** 关键字根本就没法进行分析。

我写一个 suspend 的测试方法，然后转换成 java 方法看一下，这个 suspend 函数是个啥。

写一个 Top Level 的Suspend.kt文件（在文章最后我会给出源码，一看就明白）

在文件中写了一个测试的 suspend 函数

```kotlin
suspend fun test(name: String) {

}
```

我们通过 Android Studio 再带的工具，如下图：把 Kotlin 方法转成 Java 方法

![](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/%E4%BC%98%E7%A7%80%E5%BC%80%E6%BA%90%E5%BA%93/images/1-1.jpg)

**点这个按钮**

![](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/%E4%BC%98%E7%A7%80%E5%BC%80%E6%BA%90%E5%BA%93/images/1-2.jpg)

结果如下

```java
public final class SuspendKt {
   @Nullable
   public static final Object test(@NotNull String name, @NotNull Continuation $completion) {
      return Unit.INSTANCE;
   }
}
```

看到了，我们的 suspend 的关键字，变成了 `test` 方法的一个Continuation参数，且为最后一个参数

看一下这个**Continuation类**，**记住这个类**，下面在分析的时候会遇到

```kotlin
@SinceKotlin("1.3")
public interface Continuation<in T> {
    public val context: CoroutineContext
    public fun resumeWith(result: Result<T>)
}
```

好目前的准备工作都已经完成，开始分析 **Retrofit 是怎么支持 Kotlin 协程的挂起函数的。**

#### 10.3.Retrofit 是怎么支持 Kotlin 协程的挂起函数的。

经过前面的源码解读，我们知道，最终会调用到 HttpServiceMethod 的 `parseAnnotations` 方法

##### 10.3.1.我们再看下这个方法，这次只看有关**协程**的部分

```java
//HttpServiceMethod.java
static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
    Retrofit retrofit, Method method, RequestFactory requestFactory) {
 	//1 获取 isKotlinSuspendFunction 的值，这个会在下面具体分析
  boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
  boolean continuationWantsResponse = false;
  boolean continuationBodyNullable = false;

  Annotation[] annotations = method.getAnnotations();
  Type adapterType;
  //2 如果是 Kotlin 挂起函数
  if (isKotlinSuspendFunction) {
    Type[] parameterTypes = method.getGenericParameterTypes();
    Type responseType =
        Utils.getParameterLowerBound(
            0, (ParameterizedType) parameterTypes[parameterTypes.length - 1]);
    if (getRawType(responseType) == Response.class && responseType instanceof ParameterizedType) {

      responseType = Utils.getParameterUpperBound(0, (ParameterizedType) responseType);
      //3 continuationWantsResponse 赋值为 true
      continuationWantsResponse = true;
    } else {
      
    }

    adapterType = new Utils.ParameterizedTypeImpl(null, Call.class, responseType);
    annotations = SkipCallbackExecutorImpl.ensurePresent(annotations);
  } else {
    adapterType = method.getGenericReturnType();
  }

  Converter<ResponseBody, ResponseT> responseConverter =
      createResponseConverter(retrofit, method, responseType);

  okhttp3.Call.Factory callFactory = retrofit.callFactory;
  if (!isKotlinSuspendFunction) {
    return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
   
  } else if (continuationWantsResponse) {
    //4 返回 SuspendForResponse 它是 HttpServiceMethod的子类
    return (HttpServiceMethod<ResponseT, ReturnT>)
        new SuspendForResponse<>(
            requestFactory,
            callFactory,
            responseConverter,
            (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter);
  } else {

    return (HttpServiceMethod<ResponseT, ReturnT>)
        new SuspendForBody<>(
            requestFactory,
            callFactory,
            responseConverter,
            (CallAdapter<ResponseT, Call<ResponseT>>) callAdapter,
            continuationBodyNullable);
  }
}
```

注释 1：获取 isKotlinSuspendFunction 的值，这个会在下面具体分析

注释 2：如果是 Kotlin 挂起函数，进入此代码块

注释 3：把 continuationWantsResponse 赋值为 true

注释 4：返回 SuspendForResponse 它是 HttpServiceMethod 的子类，然后看它的 `adapt`方法，这个会在下面具体分析

获取 isKotlinSuspendFunction 的值的过程

##### 10.3.2.看 requestFactory 的 isKotlinSuspendFunction 赋值

requestFactory 这个类，我们之前分析过，就是解析注解的，但是有一部分没看，就是解析方法参数上的注解，这次就看下。

```java
//RequestFactory.java
private @Nullable ParameterHandler<?> parseParameter(
    int p, Type parameterType, @Nullable Annotation[] annotations, boolean allowContinuation) {
  ParameterHandler<?> result = null;
  if (annotations != null) {
    for (Annotation annotation : annotations) {
      ParameterHandler<?> annotationAction =
        //1 遍历解析参数的注解，就是 @Path @Query @Field 等注解，具体就不看了，不是协程的重点
          parseParameterAnnotation(p, parameterType, annotations, annotation);

      if (annotationAction == null) {
        continue;
      }

      if (result != null) {
        throw parameterError(
            method, p, "Multiple Retrofit annotations found, only one allowed.");
      }

      result = annotationAction;
    }
  }

  if (result == null) {
    //2 如果是协程 ，其实比的就是最后一个值
    if (allowContinuation) {
      try {
        //3 判断参数类型是 Continuation，这个接口，前面在 10.2 小节写 Demo 时提过
        if (Utils.getRawType(parameterType) == Continuation.class) {
          // 4 isKotlinSuspendFunction 赋值为 true
          isKotlinSuspendFunction = true;
          return null;
        }
      } catch (NoClassDefFoundError ignored) {
      }
    }
    throw parameterError(method, p, "No Retrofit annotation found.");
  }

  return result;
}
```

注释 1：遍历解析参数的注解，就是 @Path @Query @Field 等注解，具体就不看了，不是协程的重点

注释 2：如果是协程 ，其实比的就是最后一个值

注释 3：判断参数类型是 Continuation，这个接口，前面在 10.2 小节写 Demo 时提过

注释 4：isKotlinSuspendFunction 赋值为 true

如果isKotlinSuspendFunction 为 true 时，返回就是 SuspendForResponse 类

接下来就要 SuspendForResponse 以及它的 `adapt` 方法了

##### 10.3.3.看一下SuspendForResponse类

```java
static final class SuspendForResponse<ResponseT> extends HttpServiceMethod<ResponseT, Object> {
  private final CallAdapter<ResponseT, Call<ResponseT>> callAdapter;

  SuspendForResponse(
      RequestFactory requestFactory,
      okhttp3.Call.Factory callFactory,
      Converter<ResponseBody, ResponseT> responseConverter,
      CallAdapter<ResponseT, Call<ResponseT>> callAdapter) {
    super(requestFactory, callFactory, responseConverter);
    this.callAdapter = callAdapter;
  }

  @Override
  protected Object adapt(Call<ResponseT> call, Object[] args) {
    //1
    call = callAdapter.adapt(call);

    //noinspection unchecked Checked by reflection inside RequestFactory.
    //2
    Continuation<Response<ResponseT>> continuation =
        (Continuation<Response<ResponseT>>) args[args.length - 1];

    // See SuspendForBody for explanation about this try/catch.
    try {
      //3
      return KotlinExtensions.awaitResponse(call, continuation);
    } catch (Exception e) {
      //4
      return KotlinExtensions.suspendAndThrow(e, continuation);
    }
  }
}
```

注释 1：调用 callAdapter 代理 call 方法

注释 2：取出最后一个参数，强转成 Continuation 类型，想想我们写的 Demo

注释 3：Call 的扩展函数（Kotlin 的写法）下面具体看下 `awaitResponse`

注释 4：出现异常，抛出异常。所以我们要在代码中，要主动 try catch，来处理错误

##### 10.3.4.看一下Call的扩展函数

```kotlin
//KotlinExtensions.kt
suspend fun <T> Call<T>.awaitResponse(): Response<T> {
  return suspendCancellableCoroutine { continuation ->
    continuation.invokeOnCancellation {
      cancel()
    }
		//调用 Call的enqueue方法
    enqueue(object : Callback<T> {
      override fun onResponse(call: Call<T>, response: Response<T>) {
        //成功回调
        continuation.resume(response)
      }

      override fun onFailure(call: Call<T>, t: Throwable) {
        //失败回调
        continuation.resumeWithException(t)
      }
    })
  }
}
```

到现在，整个用 Kotlin 协程的请求过程我们也就看完了。

### 11.总结

至此，整个 Retrofit 的整体流程就分析完了，具体细节还需要好好研究，我们再总结一下，回答开始的问题

#### 11.1.什么是动态代理？

分两点**动态**指的是在运行期，而代理指的是实现了某个接口的具体类，称之为代理，会调用了 InvocationHandler 的 `invoke`方法。

Retrofit 中的动态代理：

* 在代码运行中，会动态创建 GitHubApiService 接口的实现类，作为代理对象，代理接口的方法

* 在我们调用GitHubApiService 接口的实现类的 `listRepos`方法时，会调用了 InvocationHandler 的 `invoke`方法。
* 本质上是在运行期，生成了 GitHubApiService 接口的实现类，调用了 InvocationHandler 的 `invoke`方法。

> 具体看第 6 节

#### 11.2.整个请求的流程是怎样的

* 我们在调用 GitHubApiService 接口的 `listRepos`方法时，会调用 InvocationHandler 的 `invoke`方法
* 然后执行 `loadServiceMethod`方法并返回一个 HttpServiceMethod 对象并调用它的 `invoke`方法
* 然后执行 OkHttpCall的 `enqueue`方法
* 本质执行的是 okhttp3.Call 的 `enqueue`方法
* 当然这期间会解析方法上的注解，方法的参数注解，拼成 okhttp3.Call 需要的 okhttp3.Request 对象
* 然后通过 Converter 来解析返回的响应数据，并回调 CallBack 接口

以上就是这个Retrofit 的请求流程

#### 11.3.底层是如何用 OkHttp 请求的？

看下第 11.2小节的解释吧

> 具体看第 8 节

#### 11.4.方法上的注解是什么时候解析的，怎么解析的？

* 在 ServiceMethod.parseAnnotations(this, method); 方法中开始的
* 具体内容是在 RequestFactory 类中，进行解析注解的
* 调用 RequestFactory.parseAnnotations(retrofit, method);  方法实现的

> 具体看第 8.2 小节

#### 11.5.Converter 的转换过程，怎么通过 Gson 转成对应的数据模型的？

* 通过成功回调的 `parseResponse(rawResponse);`方法开始
* 通过 responseConverter 的 `convert` 方法
* responseConverter 是通过 converterFactories 通过遍历，根据返回值类型来使用对应的 Converter 解析

> 具体看第 8.4 小节

#### 11.6.CallAdapter 的替换过程，怎么转成 RxJava 进行操作的？

* 通过配置 addCallAdapterFactory(RxJava2CallAdapterFactory.create()) 在 callAdapterFactories 这个 list 中添加 RxJava2CallAdapterFactory
* 如果不是 Kotlin 挂起函数最终调用的是 CallAdapted 的 `adapt`方法
* callAdapter 的实例是通过 callAdapterFactories 这个 list 通过遍历，根据返回值类型来选择合适的CallAdapter

> 具体看第 9 节

#### 11.7.如何支持 Kotlin 协程的 suspend 挂起函数的？

- 通过 RequestFactory 解析方法上的参数值来判断是不是一个挂起函数，并把 isKotlinSuspendFunction 变量置为 true
- 根据 isKotlinSuspendFunction 这个变量来判断响应类型是否是 Response 类型，然后把continuationWantsResponse 置为 true
- 根据 continuationWantsResponse 这个变量，来返回 SuspendForResponse 对象
- 并调用 SuspendForResponse 的 invoke 方法
- 通过 Call 的扩展函数，来调用 Call 的 enqueue方法
- 通过协程来返回

> 具体看第 10 节

到此为止，这篇文章算写完了，当然还有很多具体细节没有研究，但对 Retrofit 的各个方面都进行了阅读。

### 12.源码地址

[RetrofitActivity.kt](https://github.com/jhbxyz/ArticleCode/blob/master/app/src/main/java/com/aboback/articlecode/retrofit/RetrofitActivity.kt)

### 13.原文地址

[一定能看懂的 Retrofit 最详细的源码解析！](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/%E4%BC%98%E7%A7%80%E5%BC%80%E6%BA%90%E5%BA%93/1Retrofit%E7%9A%84%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md)

### 14.参考资料

[Retrofit](https://square.github.io/retrofit/)

[从一道面试题开始说起 枚举、动态代理的原理](https://blog.csdn.net/lmj623565791/article/details/79278864)



