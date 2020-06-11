# View的渲染过程

**本文目标**

- View的渲染执行流程

**注意本文基于Android-28**

在上一篇文章中 ViewRootImpl 在添加 View 之前，又需要调用 requestLayout 方法，执行完整的 View 树的渲染操作。所以本文的入口，在ViewRootImpl#setView方法中的requestLayout 方法

### 1.ViewRootImpl的requestLayout方法

**ViewRootImpl#requestLayout**

```java
 @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
          //1 判断线程
            checkThread();
            mLayoutRequested = true;//2
         	//3 执行绘制流程
            scheduleTraversals();
        }
    }

	void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }

```

- 检查线程，一般会出现在子线程操作UI会报错

- 将请求布局标识符设置为 true，这个参数决定了后续是否需要执行 measure 和 layout 操作。

  

**ViewRootImpl#scheduleTraversals**

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true
      //1 插入同步屏障消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
      //2 
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

* 处向主线程消息队列中插入 SyncBarrier Message。该方法发送了一个没有 target 的 Message 到 Queue 中，在 next 方法中获取消息时，如果发现没有 target 的 Message，则在**一定的时间内跳过同步消息，优先执行异步消息**。这里通过调用此方法，**保证 UI 绘制操作优先执行**。
* 调用 Choreographer 的 postCallback 方法，实际上也是发送一个 Message 到主线程消息队列。

看一下Choreographer 的 postCallback 方法

### 2.Choreographer 的 postCallback 方法

```java
    public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }

    public void postCallbackDelayed(int callbackType,
            Runnable action, Object token, long delayMillis) {

        postCallbackDelayedInternal(callbackType, action, token, delayMillis);
    }

    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
       

        synchronized (mLock) {
    

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                // 异步消息
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
```

层层调用，最终通过 Handler 发送到 MessageQueue 中的 Message 被设置为异步类型的消息。

再看一下**mTraversalRunnable**这个参数

```java
#ViewRootImpl.java

final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

//ViewRootImpl的内部类
final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
```

**ViewRootImpl#doTraversal()**

```java
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
      //真正的开始 View 绘制流程：measure –> layout –> draw
        performTraversals();

        if (mProfile) {
            Debug.stopMethodTracing();
            mProfile = false;
        }
    }
}
```


这个方法的核心就是这三个 measure –> layout –> draw 流程

```java
private void performTraversals() {  
        ......  
        measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);
        ...
        performLayout(lp, desiredWindowWidth, desiredWindowHeight);
        ......  
        performDraw();
        }
        ......  
    }  
```

**ViewRootImpl#measureHierarchy**

```java
    private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
   
        if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
          
            if (baseSize != 0 && desiredWindowWidth > baseSize) {
              //1 
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
              //2
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
               
            }
        }


        return windowSizeMayChange;
    }

```

getRootMeasureSpec 方法获取了根 View的MeasureSpec，实际上 MeasureSpec 中的宽高此处获取的值是 Window 的宽高

**ViewRootImpl#performMeasure**

```java
   private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
        //注意mView是 DecorView，接下来是View的Measure方法
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```

* 这个 mView 就是 DecorVIew。其 DecorView 的 measure 方法中，会调用 onMeasure 方法，
* DecorView 是继承自 FrameLayout 的，因此最终会执行 FrameLayout 中的 onMeasure 方法，并递归调用子 View 的 onMeasure 方法

**ViewRootImpl#performLayout**

```java
 private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
     

        final View host = mView;
        if (host == null) {
            return;
        }
     
        try {
          //
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

            mInLayout = false;
            int numViewsRequestingLayout = mLayoutRequesters.size();
            if (numViewsRequestingLayout > 0) {
                
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);
                    mInLayout = true;
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());

                    mHandlingLayoutInLayoutRequest = false;

                }

            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }
```

最终是View的layout方法

**ViewRootImpl#performDraw**

```java
private void performDraw() {
    try {
        //draw
        boolean canUseAsync = draw(fullRedrawNeeded);
        if (usingAsyncReport && !canUseAsync) {
            mAttachInfo.mThreadedRenderer.setFrameCompleteCallback(null);
            usingAsyncReport = false;
        }
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}

 private boolean draw(boolean fullRedrawNeeded) {

        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
               //启动硬件加速绘制
                mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback);
            } else {

                //软件绘制
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
                }
            }
        }

        return useAsyncReport;
    }


```
**ViewRootImpl#drawSoftware**

````java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty, Rect surfaceInsets) {

            try {
                //1 DecroView draw
                mView.draw(canvas);

                drawAccessibilityFocusedDrawableIfNeeded(canvas);
            } finally {
                // 2
                surface.unlockCanvasAndPost(canvas);
               
            }

        return true;
    }
````

* DecorView 的 draw 方法将 UI 元素绘制到画布 Canvas 对象中

* Canvas 中的内容显示到屏幕上，实际上就是将 Canvas 中的内容提交给 SurfaceFlinger 进行合成处理。

**View#draw**

```java
public void draw(Canvas canvas) {
       

        if (!dirtyOpaque) {
            //1
            drawBackground(canvas);
        }

        if (!verticalEdges && !horizontalEdges) {
            // 2
            if (!dirtyOpaque) onDraw(canvas);

            // 3
            dispatchDraw(canvas);

            
            onDrawForeground(canvas);

            return;
        }

      .....
    }
```

*  处绘制 View 的背景；
* 处绘制 View 自身内容；
* 处表示对 draw 事件进行分发，在 View 中是空实现，实际调用的是 ViewGroup 中的实现，并递归调用子 View 的 draw 事件









