# Activity的启动流程梳理

**本文目标**

* 本文章对 startActivity 的启动流程，进行总体把握
* 本文章基于Android-28

**整体流程图**

![Activity启动流程图]([https://github.com/jhbxyz/ArticleRecord/raw/master/images/Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%9B%BE.png](https://github.com/jhbxyz/ArticleRecord/raw/master/images/Activity启动流程图.png))



### 1.startActivity方法

分析的入口，我们启动一个页面时调用startActivity方法

```kotlin
startActivity(Intent(this, DemoActivity::class.java))
```

点进去看一下

**Activity#startActivity**

```java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}
```

层层调用，最终会走到 

**Activity#startActivityForResult**

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity( //1 Instrumentation类
                    this, mMainThread.getApplicationThread(), mToken, this,//2
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            ...
        } else {
            ...
        }
    }
```

* 调用 **Instrumentation.execStartActivity** 方法。剩下的交给 Instrumentation 类去处理。Instrumentation 类主要用来监控应用程序与系统交互

* mMainThread 是 **ActivityThread** 类型，ActivityThread 可以理解为一个**进程**
* mMainThread 获取一个 **ApplicationThread** 的引用，这个引用就是用来实现进程间通信的

### 2.Instrumentation类

至此，启动就交给了Instrumentation类的execStartActivity方法了

**Instrumentation#execStartActivity**

```java
 public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
       
        ...

        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManager.getService() //1获取AMS
                .startActivity(whoThread, who.getBasePackageName(), intent,//2调用AMS方法
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);//3 检查启动Activity的结果，会抛异常
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }

```

* ActivityManager.getService().startActivity方法
* ActivityManger.getService 获取 **AMS** 的实例

看一下代码，单例模式对外提供AMS其实是一个Binder

**ActivityManager#getService**

```java
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();//获取AMS
}

private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
```

这里就是通过 AIDL 来调用 AMS 的 startActivity 方法

**startActivity 的工作重心成功移到了系统进程 AMS 中。**

### 3.ActivityManagerService类

**ActivityManagerService#startActivity**

```java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
  				//层层调用startActivityAsUser方法
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```

层层调用startActivityAsUser方法

**ActivityManagerService#startActivityAsUser**

```java
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
        boolean validateIncomingUser) {
    enforceNotIsolatedCaller("startActivity");

    userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    //1  获取一个ActivityStarter
    return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setMayWait(userId)
            .execute();//2 调用execute()

}
```

* mActivityStartController.obtainStarter 获取一个ActivityStarter并调用execute()
* execute()中会调用ActivityStarter的startActivityMayWait方法

注意；此时转移到ActivityStarter中

### 4.ActivityStarter类

**ActivityStarter#startActivityMayWait**

```java


 private int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
            int userId, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup) {
       
       ...

        //通过ActivityStackSupervisor 获取 ResolveInfo
        ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
                0 /* matchFlags */,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, mRequest.filterCallingUid));
       ...
        //通过ActivityStackSupervisor 获取 Activity
        ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

        ...

            final ActivityRecord[] outRecord = new ActivityRecord[1];
   					//1 startActivity
            int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                    ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                    allowPendingRemoteAnimationRegistryLookup);

        ...
            return res;
        }
    }
```

在 startActivityMayWait 方法中调用了一个重载的 startActivity 方法，

层层重载调用（想看的自己阅读源码，我这里就不贴了）

而**最终**会调用的 ActivityStarter 中的 startActivityUnchecked 方法来获取启动 Activity 的结果

**ActivityStarter#startActivityUnchecked**

```java
 private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {

        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);

        //计算启动 Activity 的 Flag 值
   			//不同的 Flag 决定了启动 Activity 最终会被放置到哪一个 Task 集合中。
        computeLaunchingTaskFlags();

        computeSourceStack();

        mIntent.setFlags(mLaunchFlags);

        ...

        // 处理 Task 和 Activity 的进栈操作。
        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
                mOptions);
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                ...
            } else {
               
                //启动栈中顶部的 Activity。
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
        } else if (mStartActivity != null) {
            mSupervisor.mRecentTasks.add(mStartActivity.getTask());
        }
        mSupervisor.updateUserStackLocked(mStartActivity.userId, mTargetStack);

        mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(), preferredWindowingMode,
                preferredLaunchDisplayId, mTargetStack);

        return START_SUCCESS;
    }
```

注意

* mTargetStack.startActivityLocked方法

*  mSupervisor.resumeFocusedStackTopActivityLocked方法

此时转移到ActivityStack中

### 5.ActivityStack类

首先看

**ActivityStack#startActivityLocked**

```java
void startActivityLocked(ActivityRecord r, ActivityRecord focusedTopActivity,
            boolean newTask, boolean keepCurTransition, ActivityOptions options) {
        TaskRecord rTask = r.getTask();
        final int taskId = rTask.taskId;
        // mLaunchTaskBehind tasks get placed at the back of the task stack.
        if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
           
            // 将 Task 和 Activity 入栈
            insertTaskAtTop(rTask, r);
        }
      
      ...
    }
```

方法中会调用 insertTaskAtTop 方法尝试将 Task 和 Activity 入栈

接下来调用mSupervisor.resumeFocusedStackTopActivityLocked方法

### 6.ActivityStackSupervisor类

**ActivityStackSupervisor#resumeFocusedStackTopActivityLocked**

```java

    boolean resumeFocusedStackTopActivityLocked() {
        return resumeFocusedStackTopActivityLocked(null, null, null);
    }

    boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!readyToResume()) {
            return false;
        }

        if (targetStack != null && isFocusedStack(targetStack)) {
          	//resumeTopActivityUncheckedLocked方法
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || !r.isState(RESUMED)) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        } else if (r.isState(RESUMED)) {
            // Kick off any lingering app transitions form the MoveTaskToFront operation.
            mFocusedStack.executeAppTransition(targetOptions);
        }

        return false;
    }
```

**注意：**targetStack.resumeTopActivityUncheckedLocked方法

调用再次转到ActivityStack中

### 7.ActivityStack类

**ActivityStack#resumeTopActivityUncheckedLocked**

```java
 boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mStackSupervisor.inResumeTopActivity) {
           
            return false;
        }

        boolean result = false;
        try {
      
            mStackSupervisor.inResumeTopActivity = true;
            //调用 resumeTopActivityInnerLocked
            result = resumeTopActivityInnerLocked(prev, options);

            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }

        return result;
    }
```

**ActivityStack#resumeTopActivityInnerLocked**

```java
 boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        
        ...
        //调用
        mStackSupervisor.startSpecificActivityLocked(next, true, true);

        ...

    }
```

最终代码又回到了 ActivityStackSupervisor 中的 startSpecificActivityLocked 方法。

### 8.ActivityStackSupervisor类

**ActivityStackSupervisor#startSpecificActivityLocked**

```java
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // 根据进程名称和 Application 的 uid 来判断目标进程是否已经创建，
        // 如果没有则代表进程未创建。
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        getLaunchTimeTracker().setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
          
                    app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                            mService.mProcessStats);
                }
                //关键方法 来执行启动 Activity 的操作
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
               ...
            }
        }

        // 处调用 AMS 创建 Activity 所在进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```

具体请看注释

接下来执行**realStartActivityLocked**方法

**ActivityStackSupervisor#realStartActivityLocked**

```java

    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {

        ...

        try {
           
            ...

                // 创建 Activity 启动事务，并传入 app.thread 参数
                final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
          			//重点
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                       
                        mergedConfiguration.getGlobalConfiguration(),   
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // 执行 Activity 启动事务 获取ClientLifecycleManager执行事务
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);

                ...

        return true;
    }

  
```

接下来调用ClientLifecycleManager scheduleTransaction方法

### 9.ClientLifecycleManager类

**ClientLifecycleManager#scheduleTransaction**

```java
 void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            transaction.recycle();
        }
    }
```

实际调用了

**ClientTransaction#schedule**

```java
public void schedule() throws RemoteException {
		//mClient是 ApplicationThread
    mClient.scheduleTransaction(this);
}
```

可以看出实际上是调用了启动事务 ClientTransaction 的 schedule 方法，而这个 transaction 实际上是在创建 ClientTransaction 时传入的 app.thread 对象，也就是 **ApplicationThread**。

**重点 mClient是什么时候赋值的？**mClient是在 ActivityStackSupervisor#realStartActivityLocked方法中

```java
ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,r.appToken);
```

获取ClientTransaction时赋值的（具体的自己看下代码就了然了）

**总结**

* 这里传入的 app.thread 会赋值给 ClientTransaction 的成员变量 mClient，ClientTransaction 会调用 mClient.scheduleTransaction(this) 来执行事务。
* 这个 app.thread 是 ActivityThread 的内部类 ApplicationThread，所以事务最终是调用 app.thread 的 scheduleTransaction 执行。

此时 AMS 通过进程间通信机制通知 ApplicationThread 执行

**回个神**

接下来就要去看 mClient.scheduleTransaction(this) 这个方法了

也就是**ApplicationThread**了

### 10.ApplicationThread类

说明一下ApplicationThread是 ActivityThread 的内部类

首先看事务执行的方法

**ApplicationThread#scheduleTransaction**

```java
   @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

好嘛，调用的是ActivityThread的scheduleTransaction方法

然鹅，你Command+F12找不到这个方法

去看下ActivityThread**继承**ClientTransactionHandler

```java
public final class ActivityThread extends ClientTransactionHandler {
```

去看父类ClientTransactionHandler

**ClientTransactionHandler#scheduleTransaction**

```java
public abstract class ClientTransactionHandler {

    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
      	// Android消息机制
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
  
}
```

调用 sendMessage 方法，向 Handler 中发送了一个 EXECUTE_TRANSACTION 的消息，并且 Message 中的 obj 就是启动 Activity 的事务对象。而这个 **Handler 的具体实现是 ActivityThread 中的 mH 对象**

内部类

**H#handleMessage**

```java

 class H extends Handler {

    public void handleMessage(Message msg) {
            
            switch (msg.what) {
                case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                		// TransactionExecutor 类
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                        transaction.recycle();
                    }
                    break;

            }

 }
```



### 11.TransactionExecutor类 

**TransactionExecutor#execute**

```java
public void execute(ClientTransaction transaction) {
    final IBinder token = transaction.getActivityToken();
		//
    executeCallbacks(transaction);

    executeLifecycleState(transaction);
    mPendingActions.clear();
}
```

**TransactionExecutor#executeCallbacks**

```java
  public void executeCallbacks(ClientTransaction transaction) {
        final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
        ....
       
        final int size = callbacks.size();
        for (int i = 0; i < size; ++i) {
            final ClientTransactionItem item = callbacks.get(i);
           
            // item 执行
            item.execute(mTransactionHandler, token, mPendingActions);
            item.postExecute(mTransactionHandler, token, mPendingActions);
           
            if (postExecutionState != UNDEFINED && r != null) {
                final boolean shouldExcludeLastTransition =
                        i == lastCallbackRequestingState && finalState == postExecutionState;
                cycleToPath(r, postExecutionState, shouldExcludeLastTransition);
            }
        }
    }

```

在 executeCallback 方法中，会遍历事务中的 callback 并执行 execute 方法，这些 callbacks 是何时被添加的呢？

**还记得 ClientTransaction 是如何创建被创建的吗？**

请看**ActivityStackSupervisor#realStartActivityLocked**这个方法

```java
clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent)...
```

加入的是LaunchActivityItem

所有看LaunchActivityItem的 execute方法

### 12.LaunchActivityItem类

```java
@Override
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client);
  	//终于看到曙光了 handleLaunchActivity
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

终于到了跟 Activity 生命周期相关的方法了，图中 client 是 ClientTransationHandler 类型，实际实现类就是 ActivityThread。因此最终方法又回到了 ActivityThread。

也就是 ActivityThread的handleLaunchActivity方法

### 13.ActivityThread类

 **ActivityThread#handleLaunchActivity**

```java
  public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
        
        ...
        //初始化WindowManagerGlobal
        WindowManagerGlobal.initialize();
				//调用 performLaunchActivity 创建并显示 Activity
        final Activity a = performLaunchActivity(r, customIntent);

        ...

        return a;
    }

```

*  WindowManagerGlobal.initialize();这个方法后面会分析 Window、Activity、View 三者之间的关系这个类很重要

**调用performLaunchActivity**

```java

    /**  Core implementation of activity launch. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        
        ...

        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //通过反射创建目标 Activity 对象
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            ...
        }

        try {
            //重要方法 仅会创建一个Application
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {

                appContext.setOuterContext(activity);

                // 调用 attach 方法建立 Activity 与 Context 之间的联系，
                // 创建 PhoneWindow 对象，并与 Activity 进行关联操作
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

               ...

                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    //通过 Instrumentation 最终调用 Activity 的 onCreate 方法。
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
              

        return activity;
    }
```

**至此，目标 Activity 已经被成功创建并执行生命周期方法**



