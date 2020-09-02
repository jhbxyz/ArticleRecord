# Android Jetpack组件ViewModel基本使用和原理分析

本文整体流程：首先要知道什么是 [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel)，然后演示一个例子，来看看 ViewModel 是怎么使用的，接着提出问题为什么是这样的，最后读源码来解释原因！

### 1.什么是ViewModel

##### 1.1.定义

[`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel) 类旨在以注重生命周期的方式存储和管理界面相关的数据。[`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel) 类让数据可在发生屏幕旋转等配置更改后继续留存。在对应的作用域内，保正只生产出对应的唯一实例，保证UI组件间的通信。

ViewModel 一般要配合 LiveData、DataBinding一起使用

##### 1.2.特点

通过定义我们可以得出

- ViewModel不会随着Activity的屏幕旋转而销毁；
- 在对应的作用域内，保正只生产出对应的唯一实例，保证UI组件间的通信

重点说一下ViewModel和onSaveInstanceState的关系

* 对于简单的数据，Activity 可以使用 `onSaveInstanceState()` 方法从 `onCreate()` 中的捆绑包恢复其数据，但此方法仅适合可以序列化再反序列化的少量数据，而不适合数量可能较大的数据，如用户列表或位图。

* ViewModel存储大量数据，不用序列化与反序列化

* onSaveInstanceState存储少量数据

* 相辅相成，不是替代

* 进程关闭是onSaveInstanceState的数据会保留，而ViewModel销毁

### 2.ViewModel的基础使用

这个例子，主要是在 打印 User 的信息，并且点击按钮的时候更新 User 的信息并打印

##### 2.1首先看一下 UserViewModel这个文件

```kotlin
//UserViewModel.kt

//自定义 User 数据类
data class User(var userId: String = UUID.randomUUID().toString(), var userName: String)

class UserViewModel : ViewModel() {

    private val userBean = User(userName = "刀锋之影")
    // 私有的 user LiveData
    private val _user = MutableLiveData<User>().apply {
        value = userBean
    }
    // 对外暴露的,不可更改 value 值的LiveData
    var userLiveData: LiveData<User> = _user

    //更新 User 信息
    fun updateUser() {
        //重新给 _user 赋值
        _user.value = userBean.apply {
            userId = UUID.randomUUID().toString()
            userName = "更新后: userName = 泰隆"
        }
    }
}
```

* 自定义 User 数据类
* 继承ViewModel，初始化 User 
* 声明私有的 user LIveData 用来更新数据
* 对外暴露的，不可更改 value 值的LiveData
* updateUser() 更新 User 信息的方法

##### 2.2.再看下ViewModelActivity的内容

```kotlin
class ViewModelActivity : AppCompatActivity() {

    //初始化 UserViewModel 通过 ViewModelProvider
    private val userViewModel by lazy { ViewModelProvider(this)[UserViewModel::class.java] }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val button = Button(this)
        setContentView(button)
        
        //观察 User 数据,并打印
        userViewModel.userLiveData.observe(this, Observer { user ->
            "User = $user".log()
        })

        //点击按钮更新 User 信息
        button.setOnClickListener {
            userViewModel.updateUser()
        }
    }
}
```

* 初始化 UserViewModel
* 观察 User 数据,并打印结果
* 点击按钮时，更新 User 信息

##### 2.3.结果日志

```log
//log 日志
User = User(userId=34c1a1a4-967e-439c-91e8-795b8c162997, userName=刀锋之影)
User = User(userId=a6d0f09c-9c01-412a-ab4f-44bef700d298, userName=更新后: userName = 泰隆)
```

##### 2.4总结：

以上就是 ViewModel 的简单使用，是配合 LiveData 的，具体 LiveData 的使用以及与原理分析，请看这篇文章

[Android Jetpack组件LiveData基本使用和原理分析](https://juejin.im/post/6867036161446969352)

通过上文可以 ViewModel 的定义以及特点，可以知道 **ViewModel在对应的作用域内，保正只生产出对应的唯一实例，保证UI组件间的通信**

我们来验证一下这个特点，我再写个例子，证明一下这个特点

### 3.验证ViewModel在对应的作用域内，保正只生产出对应的唯一实例

##### 3.1.ViewModelActivity2类

在ViewModelActivity2中通过supportFragmentManager添加两个 Fragment

```kotlin
class ViewModelActivity2 : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_view_model)
        supportFragmentManager.beginTransaction()
            .add(R.id.flContainer, FirstFragment())
            .add(R.id.flContainer, SecondFragment())
            .commit()
    }
}
```

##### 3.2.两个 Fragment

```kotlin
class FirstFragment : Fragment() {
    private val TAG = javaClass.simpleName

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        val userViewModel = ViewModelProvider(activity as ViewModelStoreOwner)[UserViewModel::class.java]
        "userViewModel = $userViewModel".logWithTag(TAG)
        return super.onCreateView(inflater, container, savedInstanceState)
    }
}
```

```kotlin
class SecondFragment : Fragment() {
    private val TAG = javaClass.simpleName
    
    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        val userViewModel = ViewModelProvider(activity as ViewModelStoreOwner)[UserViewModel::class.java]
        "userViewModel = $userViewModel".logWithTag(TAG)
        return super.onCreateView(inflater, container, savedInstanceState)
    }
}
```

* 在 FirstFragment和SecondFragment的onCreateView方法中实例化UserViewModel对象

* 其中的参数都为**activity as ViewModelStoreOwner**其实也就是ViewModelActivity2

* 打印UserViewModel对象的地址值，来看日志

##### 3.3.结果日志

```log
E/FirstFragment: userViewModel = com.jhb.awesomejetpack.viewmodel.UserViewModel@9940311
E/SecondFragment: userViewModel = com.jhb.awesomejetpack.viewmodel.UserViewModel@9940311
```

可以看到两个 Fragment 中 UserViewModel是同一个对象。

可以这两个 Fragment 可以使用其 Activity 范围共享 [`ViewModel`](https://developer.android.com/reference/androidx/lifecycle/ViewModel) 来处理此类通信

### 4.抛出问题

* ViewModel为什么不会随着Activity的屏幕旋转而销毁；
* 为什么在对应的作用域内，保正只生产出对应的唯一实例，保证UI组件间的通信
* onCleared方法在什么调用

### 5.分析源码前的准备工作

##### 5.1ViewModel 的生命周期

![](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/Jetpack/images/1-4.png)

##### 5.2.几个类的感性认识

* ViewModelStoreOwner：是一个接口，用来获取一个ViewModelStore对象
* ViewModelStore：存储多个ViewModel，一个ViewModelStore的拥有者( Activity )在配置改变， 重建的时候，依然会有这个实例
* ViewModel：一个对 Activity、Fragment 的数据管理类，通常配合 LiveData 使用

* ViewModelProvider：创建一个 ViewModel 的实例，并且在给定的ViewModelStoreOwner中存储 ViewModel

### 6.源码分析

再看上面第一个例子中的代码

```kotlin
class ViewModelActivity : AppCompatActivity() {

    //初始化 UserViewModel 通过 ViewModelProvider
    private val userViewModel by lazy { ViewModelProvider(this)[UserViewModel::class.java] }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val button = Button(this)
        setContentView(button)

        //观察 User 数据,并打印
        userViewModel.userLiveData.observe(this, Observer { user ->
            "User = $user".log()
        })

        //点击按钮更新 User 信息
        button.setOnClickListener {
            userViewModel.updateUser()
        }
    }
}
```

首先看下UserViewModel的初始化过程。

` private val userViewModel by lazy { ViewModelProvider(this)[UserViewModel::class.java] }`

注：上面代码类似数组的写法是 Kotlin 的写法，其实是 ViewModelProvider 的**get**方法

### 7.ViewModelProvider的构造方法，以及 get 方法

##### 7.1ViewModelProvider构造方法

先看ViewModelProvider构造方法，传入的参数为当前的 AppCompatActivity

```java
//ViewModelProvider.java

private final Factory mFactory;
private final ViewModelStore mViewModelStore;

public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
    this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
            ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
            : NewInstanceFactory.getInstance());
}

public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
    mFactory = factory;
    mViewModelStore = store;
}
```

* 通过 ViewModelStoreOwner获取ViewModelStore对象并给 mViewModelStore赋值
* 给mFactory赋值，这里赋值的是**NewInstanceFactory**这个对象

##### 7.2.ViewModelProvider的 get 方法

```java
//ViewModelProvider.java

private static final String DEFAULT_KEY = "androidx.lifecycle.ViewModelProvider.DefaultKey";

public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
  	//1
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}
```

注释1：

* 调用了两个参数的 get 方法

* 第一个参数是字符串的拼接，用来以后获取对应 ViewModel 实例的，保证了同一个 Key 取出是同一个 ViewModel
* 第二参数是 UserViewModel 的字节码文件对象

看下两个参数的`get`方法

```java
//ViewModelProvider.java
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);//1
		//2
    if (modelClass.isInstance(viewModel)) {
        if (mFactory instanceof OnRequeryFactory) {
            ((OnRequeryFactory) mFactory).onRequery(viewModel);
        }
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
  	//3
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
    } else {
        viewModel = (mFactory).create(modelClass);
    }
  	//4
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}
```

注释 1：从ViewModelStore中，根据 key，取一个 ViewModel，ViewModelStore源码下文分析

注释 2：判断取出来的 ViewModel 实例和传进来的是否是一个，是同一个，直接返回此缓存中实例

注释 3：通过Factory创建一个ViewModel

注释 4：把新创建的ViewModel用ViewModelStore存储起来，以备下次使用，最后返回新创建的ViewModelStore

这里看一下ViewModel是怎么通过Factory创建出来的

通过 7.1 小节可以知道，这个Factory的实例是NewInstanceFactory

##### 7.3.NewInstanceFactory的create方法

```java
//ViewModelProvider.java 中的 AndroidViewModelFactory.java
public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
    //noinspection TryWithIdenticalCatches
    try {
        return modelClass.newInstance();
    } catch (InstantiationException e) {
        throw new RuntimeException("Cannot create an instance of " + modelClass, e);
    } catch (IllegalAccessException e) {
        throw new RuntimeException("Cannot create an instance of " + modelClass, e);
    }
}
```

简单粗暴，通过反射，直接创建了ViewModel对象。

这里扩展一个，在实例UserViewModel的时候

```kotlin
private val userViewModel by lazy { ViewModelProvider(this,ViewModelProvider.AndroidViewModelFactory.getInstance(application))[UserViewModel::class.java] }
```

也可以通过两个参数的构造方法，来实例化，其中第二个参数就是Factory类型。然后就会用 AndroidViewModelFactory来实例化UserViewModel，我们来具体看下代码

AndroidViewModelFactory是NewInstanceFactory的子类

```java
//ViewModelProvider.java 中的 AndroidViewModelFactory
public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
    if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
        //noinspection TryWithIdenticalCatches
        try {
            return modelClass.getConstructor(Application.class).newInstance(mApplication);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (InstantiationException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        }
    }
    return super.create(modelClass);
}
```

如果我们创建的UserViewModel当初继承的是AndroidViewModel类就走`modelClass.getConstructor(Application.class).newInstance(mApplication);`实例化方法，否则就走父类的实例化方法，也就是**NewInstanceFactory的create方法**

在开发中建议使用**AndroidViewModel**类，它会提供给一个Application级别的 Context。

接下来看一下ViewModelStoreOwner是什么，以及它的具体实现

### 8.ViewModelStoreOwner

```java
public interface ViewModelStoreOwner {
    /**
     * Returns owned {@link ViewModelStore}
     *
     * @return a {@code ViewModelStore}
     */
    @NonNull
    ViewModelStore getViewModelStore();
}
```

* 一个接口，里面一个方法返回了ViewModelStore对象
* 它的实现类在 AndroidX 中ComponentActivity和 Fragment

ComponentActivity的关键代码

```java
//ComponentActivity.java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements ViewModelStoreOwner,XXX{
   
    private ViewModelStore mViewModelStore;

    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    } 
}
```

* 创建了一个ViewModelStore并返回了

来看下这个ViewModelStore类

### 9.ViewModelStore

##### 9.1.ViewModelStore的源码

我下面贴的是完整代码，对你没看错。

```java
public class ViewModelStore {
		//1
    private final HashMap<String, ViewModel> mMap = new HashMap<>();
		//2
    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }
		//3
    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    //4
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

注释 1 ：声明一个 Map 来存储ViewModel

注释 2：存储ViewModel，这个方法我们在7.2 小节ViewModelProvider的 `get` 方法中用到过

注释 3：取出 ViewModel，这个方法我们在7.2 小节ViewModelProvider的 `get` 方法中用到过。注意在从 Map中去 ViewModel 的时候是根据 Key，也就是7.2小节注释 1 拼接的那个字符串**DEFAULT_KEY + ":" + canonicalName**。这也就解释了第 4 节的疑问 **为什么在对应的作用域内，保正只生产出对应的唯一实例**

注释 4：这个是一个重点方法了，表明要清空存储的数据，还会调用到ViewModel的 `clear` 方法，也就是最终会调用带 ViewModel 的`onCleared()`方法



那么这个ViewModelStore的 `clear` 方法，什么时候会调用呢？

##### 9.2.ComponentActivity的构造方法

```java
//ComponentActivity.java
public ComponentActivity() {
    Lifecycle lifecycle = getLifecycle();
   	
    getLifecycle().addObserver(new LifecycleEventObserver() {
        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
          	//1
            if (event == Lifecycle.Event.ON_DESTROY) {
                if (!isChangingConfigurations()) {
                    getViewModelStore().clear();
                }
            }
        }
    });

}
```

在ComponentActivity的构造方法中，我们看到，在 Activity 的生命周期为 onDestory的时候，并且当前不是，配置更改（比如横竖屏幕切换）就会调用ViewModelStore 的 clear 方法，进一步回调用 ViewModel 的`onCleared`方法。

这就回答了第四节提出的问题**onCleared方法在什么调用**



最后看一下 ViewModel 的源码，以及其子类AndroidViewModel

### 10.ViewModel 的源码

ViewModel类其实更像是**更规范化的抽象接口**

```java
public abstract class ViewModel {
    private volatile boolean mCleared = false;

    @SuppressWarnings("WeakerAccess")
    protected void onCleared() {
    }

    @MainThread
    final void clear() {
        mCleared = true;
        if (mBagOfTags != null) {
            synchronized (mBagOfTags) {
                for (Object value : mBagOfTags.values()) {
                    // see comment for the similar call in setTagIfAbsent
                    closeWithRuntimeException(value);
                }
            }
        }
        onCleared();
    }

}
```

ViewModel 的子类AndroidViewModel

```java
public class AndroidViewModel extends ViewModel {
    @SuppressLint("StaticFieldLeak")
    private Application mApplication;

    public AndroidViewModel(@NonNull Application application) {
        mApplication = application;
    }

    /**
     * Return the application.
     */
    @SuppressWarnings({"TypeParameterUnusedInFormals", "unchecked"})
    @NonNull
    public <T extends Application> T getApplication() {
        return (T) mApplication;
    }
}
```

提供了一个规范，提供了一个 Application 的 Context

到现在整个源码过程就看了，包括前面，我们提到的那几个关键类的源码。

到目前为止，我们第 4 节抛出的问题，已经解决了，两个了，还有一个**ViewModel为什么不会随着Activity的屏幕旋转而销毁；**

### 11.分析为啥ViewModel不会随着Activity的屏幕旋转而销毁

首先知道的是 ViewModel 不被销毁，是在一个 ViewModelStore 的 Map 中存着呢，所以要保证ViewModelStore不被销毁。

首先得具备一个前置的知识

在 Activity 中提供了 `onRetainNonConfigurationInstance` 方法，用于处理配置发生改变时数据的保存。随后在重新创建的 Activity 中调用 `getLastNonConfigurationInstance` 获取上次保存的数据。

##### 11.1.onRetainNonConfigurationInstance方法

```java
//ComponentActivity.java
/**
 * Retain all appropriate non-config state.  You can NOT
 * override this yourself!  Use a {@link androidx.lifecycle.ViewModel} if you want to
 * retain your own non config state.
 */
@Override
@Nullable
public final Object onRetainNonConfigurationInstance() {
    Object custom = onRetainCustomNonConfigurationInstance();

    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
        // No one called getViewModelStore(), so see if there was an existing
        // ViewModelStore from our last NonConfigurationInstance
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            viewModelStore = nc.viewModelStore;
        }
    }

    if (viewModelStore == null && custom == null) {
        return null;
    }
		//1
    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    nci.viewModelStore = viewModelStore;
    return nci;
}
```

注意看下方法上的注释

* 不需要也不能重写此方法，因为用 final 修饰
* 配置发生改变时数据的保存，用ViewModel就行
* 注释 1：把ViewModel存储在 NonConfigurationInstances 对象中

现在再看下ComponentActivity 的 getViewModelStore方法

```java
//ComponentActivity.java
@NonNull
@Override
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mViewModelStore == null) {
      	//1
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // Restore the ViewModelStore from NonConfigurationInstances
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
    return mViewModelStore;
}
```

注释 1：获取了NonConfigurationInstances一个对象，不为空从其身上拿一个ViewModelStore，这个就是之前保存的ViewModelStore

当 Activity 重建时还会走到`getViewModelStore`方法，这时候就是在NonConfigurationInstances拿一个缓存的ViewModelStore。

而

### 12.总结

##### 12.1.ViewModel为什么不会随着Activity的屏幕旋转而销毁

主要是通过onRetainNonConfigurationInstance方法把ViewModelStore缓存在NonConfigurationInstances中，在`getViewModelStore`取出ViewModelStore。具体内容看第 11 节

##### 12.2.为什么在对应的作用域内，保正只生产出对应的唯一实例

ViewModelStore的 `get`方法，是根据 key 来取值的，如果 key相同，那取出来的ViewModel就是一个。具体内容看第 9.2 小节

##### 12.3.onCleared方法在什么调用

当 Activity 真正销毁的时候，而不是配置改变会调用ViewModelStore的 `clear`进而调用了ViewModel的`onCleared`，具体内容看第 9.2 小节







### 13.源码地址

[ViewModelActivity.kt](https://github.com/jhbxyz/AwesomeJetpack/blob/master/app/src/main/java/com/jhb/awesomejetpack/viewmodel/ViewModelActivity.kt)

### 14.原文地址

[Android Jetpack组件ViewModel基本使用和原理分析](https://github.com/jhbxyz/ArticleRecord/blob/master/articles/Jetpack/3ViewModel%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%E5%92%8C%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90.md)

### 15.参考文章

[ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel)

[Android官方架构组件ViewModel:从前世今生到追本溯源](https://juejin.im/post/6844903729414537223#heading-12)



## 系列文章

- [Android Jetpack组件Lifecycle基本使用和原理分析](https://juejin.im/post/6865568507698872328)
- [Android Jetpack组件LiveData基本使用和原理分析](https://juejin.im/post/6867036161446969352)
- [Android Jetpack组件ViewModel基本使用和原理分析](https://juejin.im/post/6867448675871195149)



