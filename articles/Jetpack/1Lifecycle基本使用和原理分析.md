# Android Jetpack组件Lifecycle基本使用和原理分析

本文主要对 [Jetpack](https://developer.android.com/jetpack) 的  [Android Architecture Components](https://developer.android.com/topic/libraries/architecture/index.html) 中的 [Lifecycle](https://developer.android.com/topic/libraries/architecture/lifecycle.html) 进行分析，首先会说明什么是 Lifecycle，然后通过一个实际的例子，来验证 Lifecycle 的特性，然后抛出问题，为啥 Lifecycle 这么厉害，这么强大！最后通过阅读Lifecycle 的源码，来一步一步解答疑问！

### 1.Lifecycle简介

**什么是[Lifecycle](https://developer.android.com/topic/libraries/architecture/lifecycle.html)**

Lifecycle提供了可用于构建生命周期感知型组件的类和接口，可以根据 Activity 或 Fragment 的当前生命周期状态自动调整其行为。

**一句话：可以感知 Activity/Fragment 的生命周期**并且可以在相应的回调事件中处理，非常方便

这样 Lifecycle 库能有效的避免内存泄漏和解决常见的 Android 生命周期难题！

### 2.Lifecycle基本用法

假设我们有一这样的需求：我们想提供一个接口可以感知Activity的生命周期，并且实现回调！用 Lifecycle 是怎么实现的？

##### 2.1.定义ILifecycleObserver接口

首先我们定义一个接口去实现 **LifecycleObserver**，然后定义方法，用上**OnLifecycleEvent**注解。

```kotlin
interface ILifecycleObserver : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    fun onCreate(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun onStart(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun onResume(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun onPause(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun onStop(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    fun onDestroy(owner: LifecycleOwner)

    @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
    fun onLifecycleChanged(owner: LifecycleOwner, event: Lifecycle.Event)

}
```

**注意：**当然你也可以不实现LifecycleObserver而是实现 **DefaultLifecycleObserver** 接口，Google官方更推荐我们使用 **DefaultLifecycleObserver** 接口

你可以在build.gradle 中依赖，然后就能使用了

```groovy
def lifecycle_version = "2.2.0"
implementation "androidx.lifecycle:lifecycle-common-java8:$lifecycle_version"
```

```kotlin
class BaseLifecycle : DefaultLifecycleObserver {
		//处理生命周期回调
}
```

##### 2.2.定义ActivityLifecycleObserver类

定义ActivityLifecycleObserver类去实现我们定义好的ILifecycleObserver接口

```kotlin
class ActivityLifecycleObserver : ILifecycleObserver {

    private var mTag = javaClass.simpleName
    override fun onCreate(owner: LifecycleOwner) {
        "onCreate ".logWithTag(mTag)
    }

    override fun onStart(owner: LifecycleOwner) {
        "onStart ".logWithTag(mTag)
    }

    override fun onResume(owner: LifecycleOwner) {
        "onResume ".logWithTag(mTag)
    }

    override fun onPause(owner: LifecycleOwner) {
        "onPause ".logWithTag(mTag)
    }

    override fun onStop(owner: LifecycleOwner) {
        "onStop ".logWithTag(mTag)
    }

    override fun onDestroy(owner: LifecycleOwner) {
        "onDestroy ".logWithTag(mTag)
    }

    override fun onLifecycleChanged(owner: LifecycleOwner, event: Lifecycle.Event) {
        "onLifecycleChanged  owner = $owner     event = $event".logWithTag(mTag)
    }

}
```

这个类中，我们在对应的生命周期方法中，打印一句Log，**方便测试**！这个类就是我们将要使用的类，它是一个观察者，可以观察Activity/Fragment的生命周期

##### 2.3.定义BaseActivity

在我们的BaseActivity中通过getLifecycle()获取一个Lifecycle，然后把我们的ActivityLifecycleObserver添加进来

```kotlin
open class BaseActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(ActivityLifecycleObserver())//1
    }
}
```

**Lifecycle是被观察者**，通过Add的方式把**LifecycleObserver这个观察者**添加进来，然后在Activity 执行到对应生命周期的时候通知观察者

**注意：**此时ActivityLifecycleObserver就可以感知Activity的生命周期了，就是这么的神奇

##### 2.4.定义LifecycleActivity类

让LifecycleActivity**继承 **BaseActivity，然后运行代码，看日志

```kotlin
class LifecycleActivity : BaseActivity()
```

LifecycleActivity来作为我们默认启动的Activity，启动LifecycleActivity然后关闭页面，来查看生命周期的日志！

##### 2.5.结果日志

```log
E/ActivityLifecycleObserver: onCreate
E/ActivityLifecycleObserver: onLifecycleChanged  owner = com.jhb.awesomejetpack.lifecycle.LifecycleActivity@56d55b5     event = ON_CREATE
E/ActivityLifecycleObserver: onStart
E/ActivityLifecycleObserver: onLifecycleChanged  owner = com.jhb.awesomejetpack.lifecycle.LifecycleActivity@56d55b5     event = ON_START
E/ActivityLifecycleObserver: onResume
E/ActivityLifecycleObserver: onLifecycleChanged  owner = com.jhb.awesomejetpack.lifecycle.LifecycleActivity@56d55b5     event = ON_RESUME
E/ActivityLifecycleObserver: onPause
E/ActivityLifecycleObserver: onLifecycleChanged  owner = com.jhb.awesomejetpack.lifecycle.LifecycleActivity@56d55b5     event = ON_PAUSE
E/ActivityLifecycleObserver: onStop
E/ActivityLifecycleObserver: onLifecycleChanged  owner = com.jhb.awesomejetpack.lifecycle.LifecycleActivity@56d55b5     event = ON_STOP
E/ActivityLifecycleObserver: onDestroy
E/ActivityLifecycleObserver: onLifecycleChanged  owner = com.jhb.awesomejetpack.lifecycle.LifecycleActivity@56d55b5     event = ON_DESTROY
```

每当**LifecycleActivity**发生了对应的生命周期改变，ActivityLifecycleObserver就会执行对应事件注解的方法，其中onLifecycleChanged的注解是**@OnLifecycleEvent(Lifecycle.Event.ON_ANY)**所以每次都会调用

**总结上面的现象：**

我们声明了一个ILifecycleObserver接口，并在方法中加入了 @OnLifecycleEvent(Lifecycle.Event.XXX)注解，在BaseActivity的onCreate方法中通过`lifecycle.addObserver(ActivityLifecycleObserver())`这行代码，然后就可以在 ActivityLifecycleObserver 对应的方法中实现对具体Activity的生命周期回调了，好神奇！为什么会是这样呢？

### 3.抛出问题

- Lifecycle是怎样感知生命周期的？
- Lifecycle是如何处理生命周期的？
- LifecycleObserver的方法是怎么回调的呢？

- 为什么LifecycleObserver可以感知到Activity的生命周期

下面就一步一步的具体分析，阅读源码，从源码中寻找答案

### 4.Lifecycle的原理分析的前置准备

在分析 Lifecycle 源码之前，我们必须先对几个**重要的类**有感性的认识，方便下面看源码！

##### 4.1.LifecycleOwner

```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

生命周期持有者，返回一个Lifecycle对象，如果你使用的是 AndroidX（也属于 Jetpack 一部分）在这Activity 、Fragment 两个类中，默认实现了 LifecycleOwner 接口

我贴下代码：

下面都是简化后的代码

```java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements LifecycleOwner,XXX {

	private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }

}
```

```java
public class Fragment implements LifecycleOwner,XXX {

	private final LifecycleRegistry mLifecycleRegistry;

    public Fragment() {
        initLifecycle();
    }

    private void initLifecycle() {
        mLifecycleRegistry = new LifecycleRegistry(this);
        //....
    }

    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }

}
```

可以看到我们 Activity 和 Fragment 中默认实现了

可以看到在 ComponentActivity 和 Fragment类默认实现了 LifecycleOwner 接口，并在中 `getLifecycle()`方法返回的是**LifecycleRegistry**对象，此时 Activity 和 Fragment类中分别持有了 Lifecycle

我们先看下 Lifecycle 类是什么

##### 4.2.Lifecycle

```java
public abstract class Lifecycle {

    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);

    @MainThread
    @NonNull
    public abstract State getCurrentState();

    public enum Event {
        
    }
    public enum State {
       
    }
}
```

在Lifecycle类中定义了**添加观察者**和**移除观察者**的方法，并定义了两个枚举类，这两个类等一下再具体说

LifecycleRegistry类是对Lifecycle这个抽象类的具体实现，可以处理多个观察者，如果你自定义 LifecycleOwner可以直接使用它。

说完了被观察者，接下来看下观察者LifecycleObserver

##### 4.3.LifecycleObserver

```java
public interface LifecycleObserver {

}
```

就是一个简单的接口，这个接口只是来标志这个是对Lifecycle的观察者，内部没有任何方法，全部都依赖于**OnLifecycleEvent注解**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface OnLifecycleEvent {
    Lifecycle.Event value();
}
```

注解的值是一个 **Lifecycle.Event** 也就是 4.2小节没有看的那两个枚举中的一个，接下来去看下Lifecycle中的两个枚举类。

##### 4.4.Lifecycle.Event和Lifecycle.State

```java
public abstract class Lifecycle {

    public enum Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY
    }

    public enum State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;
        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}
```

**Event**：定一个一些枚举常量，和 Activity、Fragment 的生命周期是一一对应的，可以**响应其生命周期**，其中多了一个ON_ANY，它是可以匹配任何事件的，Event 的使用是和LifecycleObserver配合使用的，

```java
* class TestObserver implements LifecycleObserver {
*   @OnLifecycleEvent(ON_STOP)
*   void onStopped() {}
* }
```

**State**：当前Lifecycle的自己的目前的状态，它是和Event配合使用的

**Event和State之间的关系**

如下图

![](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/Jetpack/images/1-1.jpg)

##### 4.5 总结

- LifecycleOwner：可获取Lifecycle的接口，可以再 Activity、Fragment生命周期改变时，通过LifecycleRegistry类处理对应的生命周期事件，并通知 LifecycleObserver这个观察者

- Lifecycle：是被观察者，用于存储有关组件（如 Activity 或 Fragment）的生命周期状态的信息，并允许其他对象观察此状态。
- LifecycleObserver：观察者，可以通过被**LifecycleRegistry**类通过 addObserver(LifecycleObserver o)方法注册,被注册后，LifecycleObserver便可以观察到LifecycleOwner对应的**生命周期事件**

- Lifecycle.Event：分派的生命周期事件。这些事件映射到 Activity 和 Fragment 中的回调事件。
- Lifecycle.State：Lifecycle组件的当前状态。

了解上面的基本内容，就进行具体的源码分析，通过看源码，就能知道整个流程了。

### 5.Lifecycle的源码解析

##### 5.1.分析的入口BaseActivity

在基类BaseActivity中的一行代码就能实现对应生命周期的回调

```kotlin
open class BaseActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(ActivityLifecycleObserver())//1
    }
}
```

我们先看下getLifecycle() 方法，然后在看`addObserver(ActivityLifecycleObserver())`的内容，注意这时候分成两步了，我们先看`getLifecycle()`

我们点进去这个getLifecycle()方法

##### 5.2.ComponentActivity

然后我们来到了ComponentActivity中，代码如下

```java
public class ComponentActivity extends xxx implements LifecycleOwner,xxx {//1

	private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);//2

	@Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        ReportFragment.injectIfNeededIn(this);//4
        
    }

    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;//3
    }

}
```

是不是很熟悉因为，之前我们在 **4.1 小节**已经看到过了，这里看下重点，在onCreate方法中有一行代码`ReportFragment.injectIfNeededIn(this);`

**注释4：**在onCreate方法中，看到初始化了一个ReportFragment，接下来看一下ReportFragment的源码

##### 5.3.ReportFragment

```java
public class ReportFragment extends Fragment {
	
    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        dispatchCreate(mProcessListener);
        dispatch(Lifecycle.Event.ON_CREATE);·
    }

    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onStop() {
        super.onStop();
        dispatch(Lifecycle.Event.ON_STOP);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
        // just want to be sure that we won't leak reference to an activity
        mProcessListener = null;
    }

    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
      	//1
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }
				//2
        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
}
```

可以看到在ReportFragment中的各个生命周期都调用了`dispatch(Lifecycle.Event event)` 方法，传递了不同的**Event**的值，这个就是在Activity、Fragment生命周期回调时，Lifecycle所要处理的生命周期方法。

在**dispatch(Lifecycle.Event event)**方法中最终调用了`((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);`方法

看到这儿，还记得咱们在**第 3 节**的疑问吗？到这儿就可以解答前两个问题了

**1.Lifecycle是怎样感知生命周期的？**

就是在ReportFragment中的各个生命周期都调用了`dispatch(Lifecycle.Event event)` 方法，传递了不同的**Event**的值

**2.Lifecycle是如何处理生命周期的？**

通过调用了`((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);`方法，也就是LifecycleRegistry类来处理这些生命周期。

此时，就应该看**LifecycleRegistry的handleLifecycleEvent**中的代码了

##### 5.4.LifecycleRegistry

```java
//LifecycleRegistry.java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}

```

根据当前Lifecycle.Event的值，其实也就是Activity/Fragment生命周期回调的值，来获取下一个**Lifecycle.State**的状态，也就是Lifecycle将要到什么状态

```java
//LifecycleRegistry.java
static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}

```

上面代码结合这个图看，食用效果更加

![](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/Jetpack/images/1-1.jpg)

不同的Lifecycle.Event的生命周期状态对Lifecycle.State的当前状态的取值。

继续跟代码，看下当到下一个状态时，要发生什么事情

```java
//LifecycleRegistry.java
private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        return;
    }
    mHandlingEvent = true;
    sync();//1
    mHandlingEvent = false;
}

```

**注释1： sync()方法**

然后看**LifecycleRegistry#sync()** 方法

```java
//LifecycleRegistry.java
private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                + "garbage collected. It is too late to change lifecycle state.");
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);//1
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);//2
        }
    }
    mNewEventOccurred = false;
}

```

如果没有同步过，会比较mState当前的状态和mObserverMap中的eldest和newest的状态做对比，看是往前还是往后；比如mState由STARTED到RESUMED是状态向前，反过来就是状态向后。这个是和Lifecycle生命周期有关系，但不是一个东西，具体的看上面贴的图，一目了然！

**注释2：往后**这里看下往后的代码**forwardPass(lifecycleOwner);**

然后看**LifecycleRegistry#forwardPass()** 方法

```java
//LifecycleRegistry.java
private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue()；//1
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));//2
            popParentState();
        }
    }
}
```

注释1：获取ObserverWithState实例

注释2：调用ObserverWithState的dispatchEvent方法

接下来看下ObserverWithState的代码

### 6.ObserverWithState

这个类名很直接，观察者并且带着 State，

```java
//ObserverWithState.java
static class ObserverWithState {
    State mState;
    LifecycleEventObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);//1
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);//2
        mState = newState;
    }
}
```

在看`dispatchEvent`方法之前，先看下构造，ObserverWithState是怎么初始化的？这里提一句，是在Lifecycle.addObserver(@NonNull LifecycleObserver observer);方法时候初始化的。

也就是` lifecycle.addObserver(ActivityLifecycleObserver())`

```kotlin
open class BaseActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(ActivityLifecycleObserver())//1
    }
}
```

在这里初始化的。

ObserverWithState内部包括了State和LifecycleEventObserver，LifecycleEventObserver是一个接口，它继承了LifecycleObserver接口。

**注释1：**mLifecycleObserver这个的获取的实例其实是**ReflectiveGenericLifecycleObserver**，具体的点进去看一眼就明白了，我就不贴代码了，但是得注意在实例化 ReflectiveGenericLifecycleObserver(object);时候把LifecycleObserver，传入ReflectiveGenericLifecycleObserver的构造中了，此时**ReflectiveGenericLifecycleObserver持有LifecycleObserver的实例**

**注释2：**关键代码 mLifecycleObserver.onStateChanged(owner, event)，这里其实调用的是**ReflectiveGenericLifecycleObserver的onStateChanged**方法

接下来看下ReflectiveGenericLifecycleObserver的.onStateChanged方法

### 7.ReflectiveGenericLifecycleObserver

```java
//ReflectiveGenericLifecycleObserver.java
class ReflectiveGenericLifecycleObserver implements LifecycleEventObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;//LifecycleObserver的实例
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());//1
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Event event) {
        mInfo.invokeCallbacks(source, event, mWrapped);//2
    }
}

```

**注意：mWrapped其实是LifecycleObserver的实例**，前面我已经调到了。

**注释 1：**接下来看mInfo的初始化过程，这个是**最关键的代码**了

**注意注意注意，此时我们要兵分两路先看注释 1 的代码，此时注释 2 的代码是被回调的代码**

### 8.ClassesInfoCache

```java
//ClassesInfoCache.java
CallbackInfo getInfo(Class<?> klass) {
    CallbackInfo existing = mCallbackMap.get(klass);
    if (existing != null) {
        return existing;
    }
    existing = createInfo(klass, null);//1
    return existing;
}

```

**注意**：这个klass是LifecycleObserver的字节码文件对象（LifecycleObserver.class）**字节码？反射的味道**，没错继续看下去马上就有结果了。

继续跟代码

```java
private CallbackInfo createInfo(Class<?> klass, @Nullable Method[] declaredMethods) {
   ...
     //1
    Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
    boolean hasLifecycleMethods = false;
    for (Method method : methods) {
      //2
        OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
        if (annotation == null) {
            continue;
        }
        hasLifecycleMethods = true;
      
        Class<?>[] params = method.getParameterTypes();
        int callType = CALL_TYPE_NO_ARG;
        if (params.length > 0) {
            callType = CALL_TYPE_PROVIDER;
            if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                throw new IllegalArgumentException(
                        "invalid parameter type. Must be one and instanceof LifecycleOwner");
            }
        }
      //3
        Lifecycle.Event event = annotation.value();
			//4
        if (params.length > 1) {
            callType = CALL_TYPE_PROVIDER_WITH_EVENT;
            if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
                throw new IllegalArgumentException(
                        "invalid parameter type. second arg must be an event");
            }
            if (event != Lifecycle.Event.ON_ANY) {
                throw new IllegalArgumentException(
                        "Second arg is supported only for ON_ANY value");
            }
        }
        if (params.length > 2) {
            throw new IllegalArgumentException("cannot have more than 2 params");
        }
      //5
        MethodReference methodReference = new MethodReference(callType, method);
        verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
    }
    CallbackInfo info = new CallbackInfo(handlerToEvent);
    mCallbackMap.put(klass, info);
    mHasLifecycleMethods.put(klass, hasLifecycleMethods);
    return info;
}
```

上面代码比较长，但都有用其实就是反射获取方法获取注解值的过程，我们挨个看

注释1：获取LifecycleObserver.class 声明的方法，也即是我们例子中**ILifecycleObserver**接口中声明的方法

注释2：遍历方法，获取方法上声明的**OnLifecycleEvent注解**

注释3：获取OnLifecycleEvent注解上的**value**

注释4：给**callType = CALL_TYPE_PROVIDER_WITH_EVENT** 赋值

注释5：把callType和当前的method 存储到MethodReference中，方便接下来取用

看一下MethodReference中的代码

```java
//MethodReference.java
static class MethodReference {
    final int mCallType;
    final Method mMethod;

    MethodReference(int callType, Method method) {
        mCallType = callType;
        mMethod = method;
        mMethod.setAccessible(true);
    }
}
```

好的，以上的mInfo 赋值的问题就看完了

当初在第 7 节在看注释 1 的代码是兵分两路了，现在继续看第 7 节注释 2 的代码吧

也即是就是**mInfo的invokeCallbacks方法**

**继续看ClassesInfoCache的invokeCallbacks方法**

点进去来到了 ClassesInfoCache 的 invokeCallbacks方法中

```java
//ClassesInfoCache.java
void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
    invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
    invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
            target);//ON_ANY也会调用
}

private static void invokeMethodsForEvent(List<MethodReference> handlers,
        LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
    if (handlers != null) {
        for (int i = handlers.size() - 1; i >= 0; i--) {
            handlers.get(i).invokeCallback(source, event, mWrapped);//1
        }
    }
}

```

注释 1：继续看MethodReference 的invokeCallback方法

```java
//MethodReference.java
void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
    //noinspection TryWithIdenticalCatches
    try {
        switch (mCallType) {
            case CALL_TYPE_NO_ARG:
                mMethod.invoke(target);
                break;
            case CALL_TYPE_PROVIDER:
                mMethod.invoke(target, source);
                break;
            case CALL_TYPE_PROVIDER_WITH_EVENT://1
                mMethod.invoke(target, source, event);
                break;
        }
    } catch (InvocationTargetException e) {
        throw new RuntimeException("Failed to call observer method", e.getCause());
    } catch (IllegalAccessException e) {
        throw new RuntimeException(e);
    }
}
```

看到最后是用反射调用了mMethod.invoke(target);这里的**target就是LifecycleObserver**之前解释过了

mCallType和mMethod的值分别是什么呢？就是在前面**初始化mInfo存的值**，再看下源码

```java
static class MethodReference {
    final int mCallType;
    final Method mMethod;

    MethodReference(int callType, Method method) {
        mCallType = callType;
        mMethod = method;
        mMethod.setAccessible(true);
    }

    void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
        //noinspection TryWithIdenticalCatches
        try {
            switch (mCallType) {
                case CALL_TYPE_NO_ARG:
                    mMethod.invoke(target);
                    break;
                case CALL_TYPE_PROVIDER:
                    mMethod.invoke(target, source);
                    break;
                case CALL_TYPE_PROVIDER_WITH_EVENT:
                    mMethod.invoke(target, source, event);
                    break;
            }
        } catch (InvocationTargetException e) {
            throw new RuntimeException("Failed to call observer method", e.getCause());
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }
}
```

由前面分析可以知道mCallType = CALL_TYPE_PROVIDER_WITH_EVENT，mMethod就是当时遍历时当前的方法

由于之前通过Map存储过，所以invokeCallback会被**遍历调用**，最终会反射调用对方法和注解。

当然其他mCallType的值也会被反射调用

### 9.总结

##### 在来回顾当初抛出的问题

**1.Lifecycle是怎样感知生命周期的？**

就是在ReportFragment中的各个生命周期都调用了`dispatch(Lifecycle.Event event)` 方法，传递了不同的**Event**的值

**2.Lifecycle是如何处理生命周期的？**

通过调用了`((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);`方法，也就是LifecycleRegistry类来处理这些生命周期。

**3.LifecycleObserver的方法是怎么回调是的呢？**

LifecycleRegistry的handleLifecycleEvent方法，然后会通过层层调用最后通过反射到LifecycleObserver方法上的@OnLifecycleEvent(Lifecycle.Event.XXX)注解值，来调用对应的方法

**4.为什么LifecycleObserver可以感知到Activity的生命周期**

LifecycleRegistry调用handleLifecycleEvent方法时会传递Event类型，然后会通过层层调用，最后是通过反射获取注解的值，到LifecycleObserver方法上的@OnLifecycleEvent(Lifecycle.Event.XXX)注解上对应的Event的值，注意这个值是和Activity/Fragment的生命周期的一一对应的，所以就可以感知Activity、Fragment的生命周期了。

### 10.源码地址

[ActivityLifecycleObserver.kt](https://github.com/jhbxyz/AwesomeJetpack/blob/master/app/src/main/java/com/jhb/awesomejetpack/lifecycle/ActivityLifecycleObserver.kt)

### 11.原文地址

[Android Jetpack组件Lifecycle基本使用和原理分析](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/Jetpack/1Lifecycle%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%E5%92%8C%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.md)

### 12.参考文章

[使用生命周期感知型组件处理生命周期  ](https://developer.android.com/topic/libraries/architecture/lifecycle.html)

[Android官方架构组件Lifecycle:生命周期组件详解&原理分析](https://juejin.im/post/6844903773366665223#heading-8)







