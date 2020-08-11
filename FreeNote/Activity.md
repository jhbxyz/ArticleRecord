# Activity难点

### 0.Activity和Fragment的生命周期

```kotlin
 Fragment======onAttach=========
 Fragment======onCreate=========
 Fragment======onCreateView=========
 Fragment======onViewCreated=========
 Activity======onCreate=========
 Fragment======onActivityCreated=========
 Fragment======onStart=========
 Activity======onStart=========
 Activity======onResume=========
 Fragment======onResume=========
 Fragment======onPause=========
 Activity======onPause=========
 Fragment======onStop=========
 Activity======onStop=========
 Fragment======onDestroyView=========
 Fragment======onDestroy=========
 Fragment======onDetach=========
 Activity======onDestroy=========
```

### 1.Intent.FLAG_ACTIVITY_NEW_TASK 的意思

##### 1.1报如下错误

Service中调用startActivity和在BroadcastReceiver（静态注册）中通过onReceive传递过来的context.startActivity时（该context类型为ReceiverRestrictedContext，和Service一样，都没有重写startActivity）如果不加FLAG_ACTIVITY_NEW_TASK的话会报如下错误的原因

为什么在**Activity**中不加FLAG_ACTIVITY_NEW_TASK调用startActivity时不会报错呢。原因是因为Activity重写了ContextWrapper中的startActivity方法

##### 1.2 FLAG_ACTIVITY_NEW_TASK不是开启了一个新栈

原则是：设置此状态，首先会查找是否存在和被启动的Activity具有相同的**亲和性**的任务栈（即taskAffinity，注意同一个应用程序中的activity的亲和性一样），如果有，则直接把这个栈整体移动到前台，并保持栈中的状态不变，即栈中的activity顺序不变，如果没有，则新建一个栈来存放被启动的activity。

我们是在同一个应用中进行Activity跳转的，所以它不会创建新的Task。

##### 1.3总结

- 在Activity上下文之外启动Activity需要给Intent设置FLAG_ACTIVITY_NEW_TASK标志，不然会报异常。
- 加了该标志，如果在同一个应用中进行Activity跳转，不会创建新的Task，只有在不同的应用中跳转才会创建新的Task

### 2.taskAffinity

- taskAffinity属性对standard和singleTop模式没有影响









### setResult()的调用时机

因为在 Activity B 退回Activity A过程中，执行过程是

　　B---onPause
　　A---onActivityResult
　　A---onRestart
　　A---onStart
　　A---onResume
　　B---onStop
　　B---onDestroy



**setResult()应该在什么时候调用呢**？从源码可以看出，Activity返回result是在被finish的时候，也就是说调用setResult()方法必须在finish()之前。