

# Activity、Window、View的层级结构

**本文目标**

* Activity、Window、View的层级结构
* setContentView的源码解析
* Activity的初始化、PhoneWindow初始化、DecorView的初始化
* 系统的布局源码，自己的layout布局加载到哪儿了

**注意本文基于Android-28**

先来一张结构清晰的结构图，一目了然！

![Activity的启动流程.png](https://raw.githubusercontent.com/jhbxyz/ArticleRecord/master/images/2-1.png)

**注意：**

* layoutResource是一个线性布局，是加载的系统布局，也就是root

### 1.Activity的setContentView方法

本文的入口，Activity的setContentView方法来分析

**Activity#setContentView**

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);//1 调用的是Window的方法
    initWindowDecorActionBar();
}
```

 getWindow()

```java
private Window mWindow; //PhoneWindow
public Window getWindow() {
    return mWindow;
}
```

**mWindow是什么时候赋值的？**

**Activity#attach**

```java
 final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context);
   			//1 初始化 PhoneWindow
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);//2 回调，注意这个方法

        ...
				//UiThread
        mUiThread = Thread.currentThread();

        ...

        //3设置WindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        
        ....
    }

```

注释3方法调用

```java
#Window.setWindowManager
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    mAppToken = appToken;
    mAppName = appName;
    mHardwareAccelerated = hardwareAccelerated
            || SystemProperties.getBoolean(PROPERTY_HARDWARE_UI, false);
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
  	//是实现类 WindowManagerImpl
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}


#WindowManagerImpl.createLocalWindowManager
public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
}


```

**总结**

* 初始化了PhoneWindow
* 设置了setCallback **this**是当前的Activity **此时PhoneWindow持有Activity的实例**
* PhoneWindow此时持有了WindowManagerImpl的实例



在Activity的attach的方法中给**mWindow**赋值，此时**Activity持有PhoneWindow的实例**

**问题又来了那么attach方法是什么时候调用的？**

在Activity的启动流程中， startActivity 的过程，我们可以知道经过层层调用，最终代码会调用到 ActivityThread 中的 performLaunchActivity 方法，通过反射创建 Activity 对象，并执行其 attach 方法。

回顾一下ActivityThread#performLaunchActivity

### 2.ActivityThread的performLaunchActivity方法

**ActivityThread#performLaunchActivity**

```java
 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
    
        ...
				// ContextImpl 创建Activity所需的Context
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //1 Activity通过ClassLoader创建出来
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
          
        } catch (Exception e) {
           
        }

        try {
          	//初始化Application
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);


            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
               
               ...
								//调用attach方法
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

              ...

                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                  	//调用Activity的oncreate方法
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                

                r.activity = activity;
            }
          
      	 		...

        return activity;
    }
```

好了，看到这儿，已经了然了

* Activity的初始化
* Window的初始化
* WindowManager的初始化

**总结：**

* ActivityThread调用performLaunchActivity方法初始化**Activity**的实例
* Activity调用的attach初始化Window(也就是PhoneWindow)给**mWindow**赋值，此时**Activity持有PhoneWindow的实例**
* PhoneWindow调用setCallback方法，**此时PhoneWindow持有Activity的实例**
* PhoneWindow调用setWindowManager方法**此时PhoneWindow持有WindowManagerImpl的实例**



接下来继续分析 getWindow().setContentView(layoutResID)方法

### 3.Window的setContentView方法

**Window#setContentView**

其实也就是PhoneWindow的方法，PhoneWindow是Window的唯一子类

```java
 public void setContentView(int layoutResID) {

        if (mContentParent == null) {
            //初始化DecorView
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            //把自己的布局加载到mContentParent中
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            //内容更新的回调，会回调Activity的方法
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }

```

看installDecor()方法

```java
 private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);//1 构造DecorView
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            ...
        } else {
            mDecor.setWindow(this);//2 DecorView持有Window
        }
        if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);//初始化mContentParent

        }
        ...
    }

```

```java
protected DecorView generateDecor(int featureId) {
    ...
    return new DecorView(context, featureId, this, getAttributes());
}
```
**重点看一下mContentParent的初始化**，我们自己实现的布局是被添加到 mContentParent 中的。

### 4.PhoneWindow#generateLayout

这个方法很重要

```java
  protected ViewGroup generateLayout(DecorView decor) {

        //根据当前设置的主题来加载默认布局
        TypedArray a = getWindowStyle();


        int layoutResource;
        if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss;
            setCloseOnSwipeEnabled(true);
        }  else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
           
            ...

        } else {
            // 1 系统的布局
            layoutResource = R.layout.screen_simple;
            
        }

        mDecor.startChanging();
        // 2 选择对应布局创建添加到DecorView中
        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
        
        ...

        mDecor.finishChanging();

        return contentParent;
    }
```

看一下注释1的布局 **R.layout.screen_simple** 是啥

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

* screen_simple是一个LinearLayout (垂直方向)
* 有两个子View：ViewStub和FrameLayout
* 自己的布局加载到这个FrameLayout中 (android:id="@android:id/content")

继续往下走**注释2**

##### 5.DecorView的onResourcesLoaded方法

看一下DecorView是一个啥，就是一个FrameLayout

```java
public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks {
```

**重点方法**

**DecorView#onResourcesLoaded**

```java
 void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
        
        ...
				//1 R.layout.screen_simple
        final View root = inflater.inflate(layoutResource, null);
        if (mDecorCaptionView != null) {
            
            ... 

        } else {

            //2 DecorView中添加了 root
            addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        }
   			//
        mContentRoot = (ViewGroup) root;
        initializeElevation();
    }

```

* xml转成View
* DecorView中添加了R.layout.screen_simple的布局

最后走到了

```java
mLayoutInflater.inflate(layoutResID, mContentParent);
```

**总结：**

* Window的setContentView方法会初始化DecorView
* DecorView初始化中会调用初始化mContentParent的方法
* 初始化mContentParent时，DecorView会加载系统布局是一个LinearLayout
* 最后mContentParent会加载，自己写布局也就是(Activity#setContentView(@LayoutRes int layoutResID)时的布局 )

至此，我们在 Activity 中传入的 layoutId 布局会加载到系统布局的**R.layout.screen_simple**的content中(是一个FrameLayout)，而系统布局会加载到DecorView中

**<font color=#ff0000>疑问那 DecorView 是何时被绘制到屏幕上的呢？</font>**DecorView现在只是PhoneWindow中一个引用

请看下篇文章



