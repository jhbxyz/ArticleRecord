# Window、DecorView、View的关系

是一篇文章中遗留了一个问题：**<font color=#ff0000>疑问那 DecorView 是何时被绘制到屏幕上的呢？</font>**DecorView现在只是PhoneWindow中一个引用

**本文目标**

* DecorView是如何绘制的，以及被添加的过程
* ViewRootImpl的初始化过程

- Window是如何添加的

**注意本文基于Android-28**

我们在 Activity 中传入的 layoutId 布局，最终会填充到 DecorView 中，DecorView是一个View，View的话就会被测量，布局，绘制，最后才是显示。

Activity的生命周期onResume方法，界面才可见，所以onResume 阶段才会将 PhoneWindow 中的 DecorView 真正的绘制到屏幕上。

基于以上的分析，咱们来看源码！

### 1.ActivityThread 的 handleResumeActivity方法

我们知道Activity的启动流程，最中会通过Handler发消息，调用到handleResumeActivity方法

**ActivityThread#handleResumeActivity**

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
       

        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
       

        final Activity a = r.activity;

        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            //WindowManager
            ViewManager wm = a.getWindowManager();
        
            if (r.mPreserveWindow) {
             
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    //add  DecorView
                    wm.addView(decor, l);
                } else {
                }
            }

            //IdleHandler
        Looper.myQueue().addIdleHandler(new Idler());
    }
```

WindowManger 的 addView 结果有两个：

* DecorView 被渲染绘制到屏幕上显示；
* DecorView 可以接收屏幕触摸事件

**说一下WindowManger**

这里的WindowManger是从Window中获取到的

```java
mWindowManager = mWindow.getWindowManager();
```

也就是上一篇文章中PhoneWindow中初始化的**WindowManagerImpl**实例

所以要看WindowManagerImpl的addView方法

**tips：**这里的ViewManager、WindowManager、WindowManagerImpl是一个继承关系，自己看一下源码，很简单

PhoneWindow 只是负责处理一些应用窗口通用的逻辑（设置标题栏，导航栏等）。但是真正完成把一个 View 作为窗口添加到 WMS 的过程是由 WindowManager 来完成的。

### 2.WindowManagerImpl的addView方法

**WindowManagerImpl#addView**

```java
@Override
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

mGlobal代理了addView方法

**mGlobal是个啥？**看一下WindowManagerImpl的成员变量

```java
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
```

是一个**WindowManagerGlobal**的单利，每一个进程中只有一个实例对象

然后看WindowManagerGlobal的addView方法

**WindowManagerGlobal#addView**

非常关键的方法了

```java
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        
        ...

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            
            //初始化了ViewRootImpl
            root = new ViewRootImpl(view.getContext(), display);

            ...

            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);

            try {
                //DecorView 添加到 WMS 中
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {

                throw e;
            }
        }
    }
```

* 初始化了ViewRootImpl
* 调用了ViewRootImpl的setView方法

ViewRootImpl是一个视图层次结构的顶部，它实现了View与WindowManager之间所需要的协议

此时的View操作顺序 WindowManagerImpl -> WindowManagerGlobal -> ViewRootImpl

看一下ViewRootImpl的setView方法

### 3.ViewRootImpl的setView方法

**ViewRootImpl#setView**

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;

                ...

                //View的绘制流程 measure - layout - draw 操作
                requestLayout();

                ...

                try {
                 
                    //将 DecorView 添加到 WMS 中
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
                 ...

            }
        }
    }
```

* requestLayout 是刷新布局的操作，调用此方法后 ViewRootImpl 所关联的 View 也执行 measure - layout - draw 操作，确保在 View 被添加到 Window 上显示到屏幕之前，已经完成测量和绘制操作。
* 调用 mWindowSession 的 addToDisplay 方法将 View 添加到 WMS 中。
* mWindowSession通过 WindowManagerGlobal的getWindowSession方法初始化

**WindowManagerGlobal#getWindowSession**

```java
 public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
                    InputMethodManager imm = InputMethodManager.getInstance();
                    IWindowManager windowManager = getWindowManagerService();
                  	//1 WMS的openSession方法
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            },
                            imm.getClient(), imm.getInputContext());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }
```

* sWindowSession 实际上是 IWindowSession 类型，是一个 Binder 类型，真正的实现类是 System 进程中的 Session。 AIDL 获取 System 进程中 Session 的对象。

**WindowManagerService#openSession**

```java
@Override
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
        IInputContext inputContext) {
    if (client == null) throw new IllegalArgumentException("null client");
    if (inputContext == null) throw new IllegalArgumentException("null inputContext");
    Session session = new Session(this, callback, client, inputContext);
  	// System 进程中 Session 的对象
    return session;
}
```

**com.android.server.wm.Session**

```java
class Session extends IWindowSession.Stub implements IBinder.DeathRecipient {
  final WindowManagerService mService;

  @Override
	public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
        int viewVisibility, int displayId, Rect outFrame, Rect outContentInsets,
        Rect outStableInsets, Rect outOutsets,
        DisplayCutout.ParcelableWrapper outDisplayCutout, InputChannel outInputChannel) {
    //WMS的addWindow
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId, outFrame,outContentInsets, outStableInsets, outOutsets, outDisplayCutout, outInputChannel);
	}
  
}
  
```

* mService就是 AMS
* 其实是调用的AMS的addWindow方法

Window 已经成功的被传递给了 WMS。剩下的工作就全部转移到系统进程中的 WMS 来完成最终的添加操作。

**Window的添加请求，最终是通过WindowManagerService来添加的。**

**疑问View是如何渲染的呢？**

请看下一篇文章：

