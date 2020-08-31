# Android Jetpack组件LiveData基本使用和原理分析

[LiveData](https://developer.android.com/topic/libraries/architecture/livedata.html)一般是和 ViewModel 配合使用的，但是本文就以单独使用 LiveData 作为例子单独使用，这样可以只关注 LiveData 而不被其他所干扰。

本文整体流程：首先要知道什么是 LiveData，然后演示一个例子，来看看 LiveData 是怎么使用的，接着提出问题为什么是这样的，最后读源码来解释原因！

LiveData 的源码比较简单，底层依赖了 Lifecycle，所以懂 Lifecycle 的源码是关键，我之前写过一篇

[Android Jetpack组件Lifecycle基本使用和原理分析](https://juejin.im/post/6865568507698872328#heading-12) 最好是先看这篇文章，才能更好的理解 LiveData。

### 1.什么是 LiveData

LiveData是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。

### 2.LiveData基础使用的例子

这个例子，是点击按钮通过 LiveData 来更新 TextView 的内容

如图：

![](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/Jetpack/images/1-2.jpg)

**点击按钮后**

![](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/Jetpack/images/1-3.jpg)

**具体代码**

```kotlin
class LiveDataActivity : BaseActivity() {
    private val mContent = MutableLiveData<String>()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_live_data)

        btnUpdate.setOnClickListener {
            mContent.value = "最新值是:Update"
        }

        mContent.observe(this, Observer { content ->
            tvContent.text = content
        })
    }
}
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/tvContent"
        android:layout_width="0dp"
        android:text="Hello World"
        android:layout_height="wrap_content"
        android:textColor="#f00"
        android:gravity="center"
        android:textSize="24sp"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/btnUpdate"
        android:layout_width="wrap_content"
        android:text="Update"
        android:padding="5dp"
        android:layout_height="wrap_content"
        android:textColor="#000"
        android:textSize="18sp"
        android:layout_marginTop="20dp"
        android:textAllCaps="false"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/tvContent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

默认TextView展示的是： Hello World，点击按钮后展示的是：“最新值是:Update” 。这个就是LiveData 的简单使用。

### 3.抛出问题

为什么LiveData的工作机制是这样的

* LiveData 是怎么回调的？
* LiveData 为什么可以感知生命周期？
* LiveData 可以感知生命周期，有什么用，或者说有什么优势？
* LiveData 为什么只会将更新通知给活跃的观察者。非活跃观察者不会收到更改通知？
* LiveData此外还提供了observerForever()方法，所有生命周期事件都能通知到，怎么做到的？

解析来通过分析源码，来寻找答案。文章最后我会解释这些问题的，做一个统一的总结。

### 4.源码分析前的准备工作

我需要了解几个类，来对接下来的源码分析做一个铺垫。

先看之前例子中的代码

```kotlin
class LiveDataActivity : BaseActivity() {
    private val mContent = MutableLiveData<String>()
  	
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
				
        mContent.observe(this, Observer { content ->
            tvContent.text = content
        })
    }
}
```

只贴出了主要代码，我们来看下主要的类以及方法，方法参数

* 声明了一个MutableLiveData对象
* 调用了MutableLiveData的observe方法
* observe方法中 传入 this 和 Observer
* this 指的是LiveDataActivity对象，其实一个是一个LifecycleOwner。Observer是一个接口

来分别看下具体内容。

##### 4.1.MutableLiveData类

```java
public class MutableLiveData<T> extends LiveData<T> {

    public MutableLiveData(T value) {
        super(value);
    }

    public MutableLiveData() {
        super();
    }

    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```

* 继承了LiveData是一个可变的LiveData
* 是一个被观察者，是一个数据持有者
* 提供了 setValue 和 postValue方法，其中postValue可以在子线程调用
* postValue方法，我们下面会具体分析

##### 4.2.MutableLiveData的observe方法参数中的 this

当前 Activity 的对象，本质上是一个**LifecycleOwner** 我在这篇  [Android Jetpack组件Lifecycle基本使用和原理分析](https://juejin.im/post/6865568507698872328)中有分析过，它的源码。

```java
public interface LifecycleOwner {
    @NonNull
    Lifecycle getLifecycle();
}
```

##### 4.3MutableLiveData的observe方法参数中的 Observer

```java
public interface Observer<T> {
    /**
     * Called when the data is changed.
     * @param t  The new data
     */
    void onChanged(T t);
}
```

* Observer是一个观察者
* Observer中有一个回调方法，在 LiveData 数据改变时会回调此方法

通过以上简单分析，我们大概了解了这个几个类的作用，接下来我们一步一步看源码，来从源码中解决我们在第 3 节提出的问题。

### 5.源码分析

首先我们上面示例中的 LiveData.observe()方法开始。

```kotlin
//LiveDataActivity.kt
private val mContent = MutableLiveData<String>()

mContent.observe(this, Observer { content ->
    tvContent.text = content
})
```

我们点进observe方法中去它的源码。

### 6.LiveData类

在LiveData的observe方法中

```kotlin
//LiveData.java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
  	//1
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
  	//2
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
  	//3
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
  	//4
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
  	//5
    owner.getLifecycle().addObserver(wrapper);
}
```

**注释 1**：首先会通过LifecycleOwner获取Lifecycle对象然后获取Lifecycle 的State，如果是DESTROYED直接 return 了。忽略这次订阅

**注释 2** ：把LifecycleOwner和Observer包装成LifecycleBoundObserver对象，至于为什么包装成这个对象，我们下面具体讲，**而且这个是重点**。

**注释 3**：把观察者存到 Map 中

**注释 4**：之前添加过LifecycleBoundObserver，并且LifecycleOwner不是同一个，就抛异常

**注释 5**：通过Lifecycle和添加 LifecycleBoundObserver观察者，形成订阅关系

**总结：**

到现在，我们知道了LiveData的observe方法中会判断 Lifecycle 的生命周期，会把LifecycleOwner和Observer包装成LifecycleBoundObserver对象，然后 `Lifecycle().addObserver(wrapper) `Lifecycle 这个被观察者会在合适的实际通知观察者的回调方法。

等等，什么时候通知，咋通知的呢？这个具体流程是啥呢？

回个神，我再贴下开始的示例代码。

```kotlin
class LiveDataActivity : BaseActivity() {
    private val mContent = MutableLiveData<String>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_live_data)
				//1
        btnUpdate.setOnClickListener {
            mContent.value = "最新值是:Update"
        }

        mContent.observe(this, Observer { content ->
            tvContent.text = content
        })
    }
}
```

在点击按钮的时候 LiveData会调用setValue方法，来更新最新的值，这时候我们的观察者Observer就会收到回调，来更新 TextView。

所以接下来我们先看下 LiveData的setValue方法做了什么，LiveData还有一个postValue方法，我们也一并分析一下。

### 7.LiveData的setValue方法和postValue方法

##### 7.1.先看setValue方法

```java
//LiveData.java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);//1
}
```

调用了**dispatchingValue**方法，继续跟代码

```java
//LiveData.java
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
          	//1
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
              	//2
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
```

不管如何判断，都是调用了`considerNotify()`方法

```java
//LiveData.java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    observer.mObserver.onChanged((T) mData);//1
}
```

最终调用了`observer.mObserver.onChanged((T) mData)`方法，这个observer.mObserver就是我们的 Observer接口，然后调用它的`onChanged`方法。

**到现在整个被观察者数据更新通知观察者这个流程就通了。**

##### 7.2.然后再看下postValue方法

子线程发送消息通知更新 UI，**嗯？Handler 的味道**，我们具体看下代码

```java
//LiveData.java
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);//1
}
```

可以看到一行关键代码`ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);`

点 **postToMainThread **方法进去看下

```java
//ArchTaskExecutor.java
private TaskExecutor mDelegate;

@Override
public void postToMainThread(Runnable runnable) {
    mDelegate.postToMainThread(runnable);//1
}
```

看到 mDelegate 是 TaskExecutor对象，现在目标是看下 mDelegate 的具体实例对象是谁

```java
//ArchTaskExecutor.java
private ArchTaskExecutor() {
    mDefaultTaskExecutor = new DefaultTaskExecutor();
    mDelegate = mDefaultTaskExecutor;//1
}
```

好的，目前的重点是看下**DefaultTaskExecutor**是个啥，然后看它的`postToMainThread`方法

```java
//DefaultTaskExecutor.java
private volatile Handler mMainHandler;

@Override
public void postToMainThread(Runnable runnable) {
    if (mMainHandler == null) {
        synchronized (mLock) {
            if (mMainHandler == null) {
                mMainHandler = createAsync(Looper.getMainLooper());//1
            }
        }
    }

    mMainHandler.post(runnable);//2
}
```

注释 1：实例了一个 Handler 对象，注意构造参数 **Looper.getMainLooper()**是主线的 Looper。那么就可做到线程切换了。

注释 2：调用post 方法。

下面看下这个 Runnable 

`ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);`

这里面的方法参数是mPostValueRunnable是个 Runnable，我们看下代码

```java
//LiveData.java
private final Runnable mPostValueRunnable = new Runnable() {
    @SuppressWarnings("unchecked")
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        setValue((T) newValue);//1
    }
};
```

**注意：**postValue方法其实最终调用也是setValue方法，然后和setValue方法走的流程就是一样的了，这个上面已经分析过了。详情请看 7.1 小节

但是我们还不知道ObserverWrapper是啥，好那么接下来，我们的重点来了

我们要详细看一下LifecycleBoundObserver类了，它包装了LifecycleOwner和Observer，这就是接下来的重点内容了。

### 8.LifecycleBoundObserver类

再贴下一下代码，当LiveData调用observe方法时

```java
//LiveData.java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
  	//1
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```

注释 1 ：用LifecycleBoundObserver对LifecycleOwner 和 Observer进行了包装

##### 8.1.来看下LifecycleBoundObserver类，它是LiveData的内部类

```java
//LiveData.java
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }
}
```

两个参数，一个 owner被成员变量mOwner存储，observer参数被ObserverWrapper的 mObserver存储。

* LifecycleEventObserver是LifecycleObserver的子接口里面有一个onStateChanged方法，这个方法会在 Activity、Fragment 生命周期回调时调用，如果这么说不明看下这篇文章[Android Jetpack组件Lifecycle基本使用和原理分析](https://juejin.im/post/6865568507698872328#heading-12) 
* ObserverWrapper 是Observer包装类

我们接下来看下ObserverWrapper类

##### 8.2.ObserverWrapper类

```java
private abstract class ObserverWrapper {
    final Observer<? super T> mObserver;
    boolean mActive;
    int mLastVersion = START_VERSION;
		//1
    ObserverWrapper(Observer<? super T> observer) {
        mObserver = observer;
    }
		//2
    abstract boolean shouldBeActive();

    boolean isAttachedTo(LifecycleOwner owner) {
        return false;
    }

    void detachObserver() {
    }

    void activeStateChanged(boolean newActive) {
        if (newActive == mActive) {
            return;
        }
        // immediately set active state, so we'd never dispatch anything to inactive
        // owner
        mActive = newActive;
        boolean wasInactive = LiveData.this.mActiveCount == 0;
        LiveData.this.mActiveCount += mActive ? 1 : -1;
        if (wasInactive && mActive) {
          	//3
            onActive();
        }
        if (LiveData.this.mActiveCount == 0 && !mActive) {
          	//4
            onInactive();
        }
        if (mActive) {
          	//5
            dispatchingValue(this);
        }
    }
}
```

注：活跃状态指的是 Activity、Fragment 等生命周期处于活跃状态

注释 1：获取了我们的 Observer 对象，存储在 成员变量mObserver身上

注释 2：抽象方法，当前是否是活跃的状态

注释 3：可以继承 LiveData 来达到扩展 LiveData 的目标，并且是在活跃的状态调用

注释 4：可以继承 LiveData 来达到扩展 LiveData 的目标，并且是在非活跃的状态调用

注释 5：活跃状态，发送最新的值，来达到通知的作用， **dispatchingValue(this)方法**咋这么眼熟，对之前在 LiveData 调用 setValue 方法时，最终也会调用到此方法。那ObserverWrapper类中的**dispatchingValue**这个方法是在`activeStateChanged`方法中调用，那`activeStateChanged`啥时候调用呢？

我来看下ObserverWrapper的子类也就是最重要的那个类LifecycleBoundObserver，现在看它的完整代码

##### 8.3.LifecycleBoundObserver完整代码

这里是关键代码了

```java
//LiveData.java
class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    @NonNull
    final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }
  
    @Override
    boolean shouldBeActive() {
				//1
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        //2
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
      	//3
        activeStateChanged(shouldBeActive());
    }
		
    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }
		
    @Override
    void detachObserver() {
     	 //4
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

**注释 1**：判断当前的 Lifecycle 的生命周期是否是活跃状态，会在回调观察则 Observer 的时候进行判断，只有在活跃状态，才会回调观察者Observer的onChanged方法。

直接就回答了我们上面的这个问题**LiveData 为什么只会将更新通知给活跃的观察者。非活跃观察者不会收到更改通知？** 首先会通过LifecycleOwner获取Lifecycle对象然后获取Lifecycle 的State，并且状态大于STARTED。这里的State是和 Activity、Fragment 的生命周期是对应的，具体看这篇文章 [Android Jetpack组件Lifecycle基本使用和原理分析](https://juejin.im/post/6865568507698872328) 的第4.4小节，有详细的解释。

**注释 2**：onStateChanged每次 Activity、Fragment的生命周期回调的时候，都会走这个方法。

获取Lifecycle对象然后获取Lifecycle 的State如果为DESTROYED则移除观察者，在 Activity、Fragment的生命周期走到 onDestroy 的时候，就会取消订阅，避免内存泄漏。

**注释 3**：调用父类ObserverWrapper 的`activeStateChanged`方法，层层调用到观察者Observer的onChanged方法。（自己看下源码一目了然）

**重点来了：**在LiveData 调用`setValue`方法时，会回调观察者Observer的onChanged方法，Activity、Fragment的生命周期变化的时候**且为活跃**也会回调观察者Observer的onChanged方法。这就是为什么你在ActivityB页面，调用`setValue`方法，更新了value，在ActivityA 重新获取焦点时也同样会收到这个最新的值。

**注释 4**：移除观察者Observer，解除订阅关系。

到这个时候,LiveData 的 `observer方法`、`setValue`方法，整个流程就分析完了。

如果我们想不管生命周期，而是想在setValue的值发生改变的时候就能接受到通知，LiveData 还提供了一个`observeForever`方法

```kotlin
class LiveDataActivity : BaseActivity() {
    private val mContent = MutableLiveData<String>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_live_data)

        btnUpdate.setOnClickListener {
            mContent.value = "最新值是:Update"
        }

        //只要在值发生改变时,就能接收到
        mContent.observeForever { content ->
            tvContent.text = content
        }
    }
}
```

##### 8.4LiveData 的observeForever方法

这个方法比observe方法少一个LifecycleOwner参数，为啥呢？因为这个方法不需要感知生命周期，需要在setValue 值更新时立马收到回调。

来看下具体代码

```java
//LiveData.java
@MainThread
public void observeForever(@NonNull Observer<? super T> observer) {
    assertMainThread("observeForever");
  	//1
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing instanceof LiveData.LifecycleBoundObserver) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    wrapper.activeStateChanged(true);
}
```

注释 1 ：这里用到的是**AlwaysActiveObserver**而 `observe`方法用到是**LifecycleBoundObserver**

看一下这个AlwaysActiveObserver

```java
//LiveData.java
private class AlwaysActiveObserver extends ObserverWrapper {

    AlwaysActiveObserver(Observer<? super T> observer) {
        super(observer);
    }

    @Override
    boolean shouldBeActive() {
        return true;
    }
}
```

代码非常的简洁，在`shouldBeActive`方法中，直接 return true，这也太秀了吧

为啥直接返回 true 呢？因为这里不用管生命周期，永远都是活跃状态，所以这个方法叫**observeForever**

LiveData 的源码非常值得读，而且量不是很大，里面有许多值得学习的地方。

### 9.使用 LiveData 的优势

这个是Google官方总结的

使用 LiveData 具有以下优势：

- **确保界面符合数据状态**

  LiveData 遵循观察者模式。当生命周期状态发生变化时，LiveData 会通知 [`Observer`](https://developer.android.com/reference/androidx/lifecycle/Observer) 对象。您可以整合代码以在这些 `Observer` 对象中更新界面。观察者可以在每次发生更改时更新界面，而不是在每次应用数据发生更改时更新界面。

- **不会发生内存泄漏**

  观察者会绑定到 [`Lifecycle`](https://developer.android.com/reference/androidx/lifecycle/Lifecycle) 对象，并在其关联的生命周期遭到销毁后进行自我清理。

- **不会因 Activity 停止而导致崩溃**

  如果观察者的生命周期处于非活跃状态（如返回栈中的 Activity），则它不会接收任何 LiveData 事件。

- **不再需要手动处理生命周期**

  界面组件只是观察相关数据，不会停止或恢复观察。LiveData 将自动管理所有这些操作，因为它在观察时可以感知相关的生命周期状态变化。

- **数据始终保持最新状态**

  如果生命周期变为非活跃状态，它会在再次变为活跃状态时接收最新的数据。例如，曾经在后台的 Activity 会在返回前台后立即接收最新的数据。

- **适当的配置更改**

  如果由于配置更改（如设备旋转）而重新创建了 Activity 或 Fragment，它会立即接收最新的可用数据。

- **共享资源**

  您可以使用单一实例模式扩展 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData) 对象以封装系统服务，以便在应用中共享它们。`LiveData` 对象连接到系统服务一次，然后需要相应资源的任何观察者只需观察 `LiveData` 对象。如需了解详情，请参阅[扩展 LiveData](https://developer.android.com/topic/libraries/architecture/livedata#extend_livedata)。

### 10.总结

开始回答第 3 节的抛出的问题

##### 10.1.LiveData 是怎么回调的？

* LiveData通过`observe`或者`observeForever`方法订阅了一个观察者
* LiveData 通过调用`setValue`或`postValue`方法时，会取出观察者，调用它的`onChanged`方法
* 当然，当 Activity、Fragment 生命周期由非活跃变化为活跃状态，也会收到最新的值回调`onChanged`方法，注意这个对应的是LiveData的`observe`方法。

##### 10.2.LiveData 为什么可以感知生命周期？

* 是因为LifecycleBoundObserver类
* 以及在`observe`方法中调用了 `owner.getLifecycle().addObserver(wrapper);`这行代码，具体的看下上面的源码分析吧

##### 10.3.LiveData 可以感知生命周期，有什么用，或者说有什么优势？

* 可以自动取消订阅
* 防止内存泄漏

##### 10.4.LiveData 为什么只会将更新通知给活跃的观察者。非活跃观察者不会收到更改通知？

* 在每次调用`setValue`方法时，最走到LifecycleBoundObserver的shouldBeActive这个方法的判断上
* 这个方法返回的是状态为STARTED之后的状态才会走通知观察者回调的逻辑，否则就不执行，具体的看下上面的源码

##### 10.5.LiveData此外还提供了observerForever()方法，所有生命周期事件都能通知到，怎么做到的？

* 看 8.4 小节，有详细的分析。
* 主要AlwaysActiveObserver的shouldBeActive这个方法直接返回的 true



### 11.源码地址

[LiveDataActivity.kt](https://github.com/jhbxyz/AwesomeJetpack/blob/master/app/src/main/java/com/jhb/awesomejetpack/livedata/LiveDataActivity.kt)

### 12.原文地址

[Android Jetpack组件Lifecycle基本使用和原理分析](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/Jetpack/1Lifecycle%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%E5%92%8C%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.md)

### 13.参考文章

[LiveData](https://developer.android.com/topic/libraries/architecture/livedata)

[Android官方架构组件LiveData: 观察者模式领域二三事](https://juejin.im/post/6844903748574117901#heading-5)



