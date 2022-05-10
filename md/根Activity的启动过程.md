### 根Activity的启动过程 

根Activity的启动整体过程如下：

主要分为四部分

+ Launcher请求ATMS创建根Activity
+ ATMS会去请求zygote创建应用程序进程
+ zygote去创建应用程序进程
+ ATMS请求ApplicationThread创建根Activity

![image-20220113120442747](%E6%A0%B9Activity%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.assets/image-20220113120442747.png)

我们分析的话，不会按照上面的每一个步骤去分析。我们会按照下面三个部分去进行源码（android 10）的分析

+ Launcher请求ATMS的过程
+ ATMS到ApplicationThread的调用过程
+ ActivityThread启动Activity的过程

#### Launcher请求ATMS的过程

首先，点击桌面图标，Launcher会去调用`startActivitySafely`这个函数

```java
//Launcher.java 
public boolean startActivitySafely(View v, Intent intent, ItemInfo item,
            @Nullable String sourceContainer) {
        ...
        boolean success = super.startActivitySafely(v, intent, item, sourceContainer);//1
		...
        return success;
    }
```

有一行比较关键的代码是，他会去调用父类的`startActivitySafely`，它最终调用的是`BaseDraggingActivity`里面的方法

```java
//BaseDraggingActivity.java 
public boolean startActivitySafely(View v, Intent intent, @Nullable ItemInfo item,
            @Nullable String sourceContainer) {
		...
        // 注释1 Prepare intent
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        if (v != null) {
            intent.setSourceBounds(getViewBounds(v));
        }
        try {
            boolean isShortcut = (item instanceof WorkspaceItemInfo)
                    && (item.itemType == Favorites.ITEM_TYPE_SHORTCUT
                    || item.itemType == Favorites.ITEM_TYPE_DEEP_SHORTCUT)
                    && !((WorkspaceItemInfo) item).isPromise();
            if (isShortcut) {
                // Shortcuts need some special checks due to legacy reasons.
                startShortcutIntentSafely(intent, optsBundle, item, sourceContainer);
            } else if (user == null || user.equals(Process.myUserHandle())) {
                // Could be launching some bookkeeping activity
                if (TestProtocol.sDebugTracing) {
                    android.util.Log.d(TestProtocol.NO_START_TAG,
                            "startActivitySafely 2");
                }
                // 注释2
                startActivity(intent, optsBundle);
                ...
            }
        return false;
    }
```

代码注释1处，这里的意思大家都懂了，就是让Activity在一个新的任务栈中去启动，然后注释2处就是调用`startActivity`。这里的代码调用的是Activity中的`startActivity`

```java
//Activity.java
@Override
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}

public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                                   @Nullable Bundle options) {
    //注释1
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
            this, mMainThread.getApplicationThread(), mToken, this,
            intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
	...
    }
}
```

可以看待注释1处，由于我们启动的是跟Activity，所以mParent==null这个判断条件是true，会进入if语句，然后if语句就调用`mInstrumentation.execStartActivity`。Instrumentation这个类主要是做啥的呢？主要是用来监视系统与应用程序的交互。所以到这里，接下来的步骤就进入Instrumentation这个类里面了。

```java
@UnsupportedAppUsage
public ActivityResult execStartActivity(
    Context who, IBinder contextThread, IBinder token, Activity target,
    Intent intent, int requestCode, Bundle options) {
	...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        // 注释1
        int result = ActivityTaskManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                           intent.resolveTypeIfNeeded(who.getContentResolver()),
                           token, target != null ? target.mEmbeddedID : null,
                           requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}

```

注意注释1处。这里是调用的ActivityTaskManager.getService()，ActivityTaskManager这是android10以后新加的一个类，

```java
//ActivityTaskManager.java	
public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }

    @UnsupportedAppUsage(trackingBug = 129726065)
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE); //注释1
                    return IActivityTaskManager.Stub.asInterface(b);//注释2
                }
            };
```

这里使用 binder IPC 调用到了 **ActivityTaskManagerService（ATMS）**中。

ATMS 是个远程服务，需要通过 ServiceManager 来获取，ServiceManager 底层最终调用的还是 Native 层的 ServiceManager，它是 Binder 的守护服务，通过它能够获取在 Android 系统启动时注册的系统服务，这其中就包含这里提到的 ATMS。获取服务的代码如下：

```java
//ActivityTaskManager.java
public static IActivityTaskManager getService() {
       return IActivityTaskManagerSingleton.get();
   }

   @UnsupportedAppUsage(trackingBug = 129726065)
   private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
           new Singleton<IActivityTaskManager>() {
               @Override
               protected IActivityTaskManager create() {
                   final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);// 注释1
                   return IActivityTaskManager.Stub.asInterface(b); //注释 2
               }
           };
```

注释 1 处通过 ServiceManager 获取对应的系统服务，也就是 IBinder 类型的 ATMS 引用。注释 2 处将它转换成 ActivityTaskManager 类型的对象，这段代码采用的是 AIDL，IActivityTaskManager.java 类是由 AIDL 工具在编译时自动生成的，IActivityTaskManager.aidl 的文件路径为 frameworks/base/core/java/android/app/IActivityTaskManager.aidl。 要实现进程间通信，服务端也就是 ATMS 只需要继承 IActivityTaskManager.Stub 类并实现相应的方法也就可以了。到这里Launcher请求ATMS的过程就讲完了。我们来看一个时序图来回顾一下整个过程。

![image-20220113143115465](%E6%A0%B9Activity%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.assets/image-20220113143115465.png)

#### ATMS到ApplicationThread的调用过程

​	讲解完成Launcher到ATMS的过程，下面，我们就需要了解ATMS到ApplicationThread的过程了。首先得清楚一点就是ATMS是运行在SystemServer进程的，不是运行在应用进程的，这时候我们应用的进程还没有起来。接着上面讲

```java
//ActivityTaskManagerService.java
public final int startActivity(IApplicationThread caller, String callingPackage,
                               Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                               int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                               resultWho, requestCode, startFlags, profilerInfo, bOptions,
                               UserHandle.getCallingUserId());
}
```

ATMS中会调用startActivityAsUser方法，这里需要注意一点就是startActivityAsUser方法比startActivity多了一个参数，为UserHandle.getCallingUserId()，ATMS根据这个参数来判断调用者的权限。

```java
//ActivityTaskManagerService.java 
public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/);
    }

    int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        // 注释1
        enforceNotIsolatedCaller("startActivityAsUser");

        //注释2
        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // 注释3
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId) //这里会将 mRequest.mayWait = true
                .execute();

    }
```

注释1处判断调用者进程是否被隔离，如果被隔离则抛出`SecurityException`异常。

注释2处检查调用者是否有权限，如果没有权限也会抛出`SecurityException`异常。

注释3出通过`ActivityStartController`来获取一个ActivityStarter，并且配置了一些参数，这里要注意setMayWait方法传入了userId，会将ActivityStarter的mayWait属性置为true，后面会用到。

ActivityStarter是Android 7.0中新加入的类，它是加载Activity的控制类，会收集所有的逻辑来决定如何将 Intent和Flags转换为Activity，并将 Activity和Task以及Stack相关联。接下来就是进入到了ActivityStarter里面的逻辑了。

```java
//ActivityStarter.java
int execute() {
    try {
        //这里是true，进入if语句
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                                        mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                                        mRequest.intent, mRequest.resolvedType,
                                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                                        mRequest.inTask, mRequest.reason,
                                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
        }
        ...
    } finally {
        onExecutionComplete();
    }
}

private int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
        Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
        IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
        int userId, TaskRecord inTask, String reason,
        boolean allowPendingRemoteAnimationRegistryLookup,
        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
    // intent中不允许携带文件描述符
    if (intent != null && intent.hasFileDescriptors()) {
        throw new IllegalArgumentException("File descriptors passed in Intent");
    }
    ··· 
    // 解析Intent
    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
            0 /* matchFlags */,
                    computeResolveFilterUid(
                            callingUid, realCallingUid, mRequest.filterCallingUid));
    ···
    // 解析Activity信息，当有多个可供选择时在这里弹出选择框
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

    synchronized (mService.mGlobalLock) {
        ··· 
        // 启动Activity
        int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
                allowBackgroundActivityStart);
        ···
        return res;
    }
}

private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
         String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
         IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
         IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
         String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
         SafeActivityOptions options,
         boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
         TaskRecord inTask, boolean allowPendingRemoteAnimationRegistryLookup,
         PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
     ···
     // 此处创建了一个ActivityRecord对象，ActivityRecord 用于描述一个Activity，并用来记录 Activity 的所有信息
     // 同时会通过ActivityRecord和intent构建appToken
     ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
             callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
             resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
             mSupervisor, checkedOptions, sourceRecord);
     if (outActivity != null) {
         outActivity[0] = r;
     }

    ......

     final int res = startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
             true /* doResume */, checkedOptions, inTask, outActivity, restrictedBgActivity);//4
     .....
     return res;
 }
```

startActivity调用了另一个startActivity的重载，继而调用到了startActivityUnchecked方法。

```java
//ActivityStarter.java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {

        // 设置初始状态值，这里的r赋值给mStartActivity
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);
        // 计算启动task的标志位Flag
        // 主要处理一些Intent Flag冲突、复用问题
        // 以及SingleInstance和SingleTask的处理
        computeLaunchingTaskFlags();
  
        // 通过sourceActivity计算sourceTask
        // 主要处理 FLAG_ACTIVITY_NEW_TASK 问题
        computeSourceStack();
  
        mIntent.setFlags(mLaunchFlags);

        // 寻找是否有可以复用的ActivityRecord
        ActivityRecord reusedActivity = getReusableIntentActivity();
        ···
        if (mReusedActivity != null) {
            ···
            // 将当前栈移至前台
            mReusedActivity = setTargetStackAndMoveToFrontIfNeeded(mReusedActivity);
　　　　　    ···
            setTaskFromIntentActivity(mReusedActivity);

            if (!mAddingToTask && mReuseTask == null) {
                resumeTargetStackIfNeeded();
                return START_TASK_TO_FRONT;
            }
        }
　　　　 ···
     　　//singleTop 或者singleInstance的处理
        if (dontStart) {
            // For paranoia, make sure we have correctly resumed the top activity.
            topStack.mLastPausedActivity = null;
            if (mDoResume) {
                mRootActivityContainer.resumeFocusedStacksTopActivities();
            }
            ActivityOptions.abort(mOptions);
            if ((mStartFlags & START_FLAG_ONLY_IF_NEEDED) != 0) {
                // We don't need to start a new activity, and the client said not to do
                // anything if that is the case, so this is it!
                return START_RETURN_INTENT_TO_CALLER;
            }
            ···
            // 向已存在的Activity分发新的Intent
            // 会回调其onNewIntent()方法
            deliverNewIntent(top);
　　    　   ···
            return START_DELIVERED_TO_TOP;
        }
  
        boolean newTask = false;
        final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTaskRecord() : null;
        // 设置对应的task
        int result = START_SUCCESS;
        if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
            newTask = true;
            // intent设置了FLAG_ACTIVITY_NEW_TASK，新建task
            result = setTaskFromReuseOrCreateNewTask(taskToAffiliate);
        } else if (mSourceRecord != null) {
            // 设置sourceRecord所在栈，即standard启动模式
            result = setTaskFromSourceRecord();
        } else if (mInTask != null) {
            // 指定了启动的taskAffinity，设置到对应的task中
            result = setTaskFromInTask();
        } else {
            // 在当前焦点的task中启动，这种情况不会发生
            result = setTaskToCurrentTopOrCreateNewTask();
        }
        if (result != START_SUCCESS) {
            return result;
        }
        ··· 
        // mDoResume由外部传入，本次流程中为true
        if (mDoResume) {
            // 使用ActivityStaskSupervisor去显示该Activity
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTaskRecord().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {

                mTargetStack.ensureActivitiesVisibleLocked(mStartActivity, 0, !PRESERVE_WINDOWS);

                mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
            } else {
                // 如果当前activity是可以获取焦点的但当前Stack没有获取焦点
                if (mTargetStack.isFocusable()
                        && !mRootActivityContainer.isTopDisplayFocusedStack(mTargetStack)) {
                    // 将对应Task移至前台
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                 // 使栈顶activity可见，即resume状态
                 mRootActivityContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);
            }
        } else if (mStartActivity != null) {
            mSupervisor.mRecentTasks.add(mStartActivity.getTaskRecord());
        }
        mRootActivityContainer.updateUserStack(mStartActivity.mUserId, mTargetStack);

        mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTaskRecord(),
                preferredWindowingMode, mPreferredDisplayId, mTargetStack);

        return START_SUCCESS;
    }
```

startActivityUnchecked方法主要处理与枝管理相关的逻辑。在之前的分析中，我们在设置的时候将Flag设置为FLAG_ACTIVITY_NEW_TASK ，也就是说会执行setTaskFromReuseOrCreateNewTask方怯， 其内部会创建个新的TaskRecord ，用来描述 Activity 任务栈。 Activity 务钱其实是一个假想的模型，并不真实存在。然后会调用resumeFocusedStacksTopActivities,resumeFocusedStacksTopActivities方法中调用ActivityStack#resumeTopActivityUncheckedLocked启动栈顶的Activity。

```java
// ActivityStack.java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.防止递归，确保只有一个Activity执行该方法
            mInResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
    }
```

mInResumeTopActivity = true;用于保证每次只有一个Activity执行resumeTopActivityUncheckedLocked()操作。然后接着调用topRunningActivityLocked

```java
// ActivityStack.java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ···
    // 获取栈顶未finish的activity
    ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
    ···
    if (next.attachedToProcess()) {
        ···
    } else {
        ···
        mStackSupervisor.startSpecificActivityLocked(next, true, true);
    }
}
```

resumeTopActivityInnerLocked方法的逻辑非常复杂，主要是判断当前是否真的需要resume栈顶Activity。这里精简了一下，当我们的栈顶activity没有绑定到任何Application的时候，就会调用ActivityStackSupervisor#startSpecificActivityLocked，我们此时恰好就是这种情况。

```java
// ActivityStackSupervisor.java
void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
    // 通过ATMS获取目标进程的WindowProcessController对象
    // WindowProcessController用于ActivityManager与WindowManager同步进程状态
    // 如果wpc为空，则表示对应进程不存在或者未启动
    final WindowProcessController wpc =
            mService.getProcessController(r.processName, r.info.applicationInfo.uid);

    boolean knownToBeDead = false;
    // 这里的hasThread表示的是IApplicationThread
    if (wpc != null && wpc.hasThread()) {
        try {
            // 启动activity
            realStartActivityLocked(r, wpc, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Exception when starting activity "
                    + r.intent.getComponent().flattenToShortString(), e);
        }

        // If a dead object exception was thrown -- fall through to
        // restart the application.
        knownToBeDead = true;
    }

    // 这里主要处理键盘锁问题
    if (getKeyguardController().isKeyguardLocked()) {
        r.notifyUnknownVisibilityLaunched();
    }

    try {
        ···
        // 如果对应App进程还未启动
        // 通过handler发送消息启动进程
        final Message msg = PooledLambda.obtainMessage(
                ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
                r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
        mService.mH.sendMessage(msg);
    } finally {
        Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```

这里我们知道，如果我们的额进程没有启动，那么需要通过发送消息去启动应用的进程，这一部分的流程我们这里先不讲。真正启动Activity的语句是realStartActivityLocked()这个方法

```java
// ActivityStackSupervisor.java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
        boolean andResume, boolean checkConfig) throws RemoteException {
            ···
            // 创建一个启动Activity事务.
            final ClientTransaction clientTransaction = ClientTransaction.obtain(
                    proc.getThread(), r.appToken);

            final DisplayContent dc = r.getDisplay().mDisplayContent;
            // 添加Callback，注意这里创建的是LaunchActivityItem
            clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                    System.identityHashCode(r), r.info,
                    // TODO: Have this take the merged configuration instead of separate global
                    // and override configs.
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
                    r.icicle, r.persistentState, results, newIntents,
                    dc.isNextTransitionForward(), proc.createProfilerInfoIfNeeded(),
                            r.assistToken));

            // 设置此次事务应该执行的最终状态
            // 此次流程将会设置为resume，表示activity应该执行到onResume状态
            final ActivityLifecycleItem lifecycleItem;
            if (andResume) {
                lifecycleItem = ResumeActivityItem.obtain(dc.isNextTransitionForward());
            } else {
                lifecycleItem = PauseActivityItem.obtain();
            }
            clientTransaction.setLifecycleStateRequest(lifecycleItem);

            // 执行事务
            mService.getLifecycleManager().scheduleTransaction(clientTransaction);
    ···
    return true;
}
```

realStartActivityLocked方法中创建了一个启动Activity事务，并利用IApplicationThread发送到目标App中。从Android 8.0开始Activity的启动将会通过事务来完成，事务将会通过目标App的IApplicationThread远程发送到目标App中，然后通过ClientLifecycleManager 来执行。至此ATMS中执行的逻辑就结束了，剩下的就是目标App的ApplicationThread来执行目标Activity的各个生命周期方法了。

#### ActivityThread启动Activity过程

在经过一系列的调用工作之后，Activity的启动工作来到了目标App中。

由上面我们知道，这一过程是从ApplicationThread接收到Binder通信传递过来的事务开始执行。ApplicationThread#scheduleTransaction调用了ActivityThread#scheduleTransaction，我们直接看ActivityThread的scheduleTransaction函数

```java
// ActivityThread.java
@Override
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}
```

ActivityThread#scheduleTransaction方法定义在其父类**ClientTransactionHandler**中。

```java
//ClientTransactionHandler.java
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```

这里调用sendMessage方法往主线程发送了一条what为**ActivityThread.H.EXECUTE_TRANSACTION**的消息，并将事务传递。

ApplicationThread运行在Binder线程，与ActivityThread通信需经过Handler，而H是ActivityThread中的一个内部类，它继承自Handler，当ActivityThread调用sendMessage方法时其实就是在调用H#sendMessage。

```java
//ActivityThread.java
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    if (DEBUG_MESSAGES) {
        Slog.v(TAG,
                "SCHEDULE " + what + " " + mH.codeToString(what) + ": " + arg1 + " / " + obj);
    }
    Message msg = Message.obtain();
    msg.what = what;
    msg.obj = obj;
    msg.arg1 = arg1;
    msg.arg2 = arg2;
    if (async) {
        msg.setAsynchronous(true);
    }
    //mH就是H类的一个实例
    mH.sendMessage(msg);
}
```

这里mH发送了一个消息，那么在H类的handlerMessage里面就会收到并处理这个消息，所以我们可以看一下H类的handlerMessage方法

```java
//ActivityThread.H.java
public void handleMessage(Message msg) {
    if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
    switch (msg.what) {
        // 省略一些case
        ···
        case EXECUTE_TRANSACTION:
            final ClientTransaction transaction = (ClientTransaction) msg.obj;
            mTransactionExecutor.execute(transaction);
            if (isSystem()) {
                // Client transactions inside system process are recycled on the client side
                // instead of ClientLifecycleManager to avoid being cleared before this
                // message is handled.
                transaction.recycle();
            }
            // TODO(lifecycler): Recycle locally scheduled transactions.
            break;
        // 省略一些case
        ···
    }
    Object obj = msg.obj;
    if (obj instanceof SomeArgs) {
        ((SomeArgs) obj).recycle();
    }
    if (DEBUG_MESSAGES) Slog.v(TAG, "<<< done: " + codeToString(msg.what));
}
```

其中使用TransactionExecutor#execute来执行事务。

TransactionExecutor是专门用于处理事务的类，用于保证事务以正确的顺序执行。

```java
//TransactionExecutor.java
public void execute(ClientTransaction transaction) {
    if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");
    // ActivityRecord中的appToken
    final IBinder token = transaction.getActivityToken();
    ···
    if (DEBUG_RESOLVER) Slog.d(TAG, transactionToString(transaction, mTransactionHandler));
    // 执行所有Callback
    executeCallbacks(transaction);
    // 执行到对应生命周期
    executeLifecycleState(transaction);
    mPendingActions.clear();
    if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
}
```

之前我们在**ActivityStackSupervisor**中创建事务并且通过**addCallback**的的方式将其添加到ClientTransaction中，现在调用executeCallbacks来执行每个Callback。

```java
//TransactionExecutor.java
public void executeCallbacks(ClientTransaction transaction) {
    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
    if (callbacks == null || callbacks.isEmpty()) {
        // 快速返回路径
        return;
    }
	...
    // 循环执行每一个item的excute    
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Resolving callback: " + item);
        final int postExecutionState = item.getPostExecutionState();
        final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                item.getPostExecutionState());
        if (closestPreExecutionState != UNDEFINED) {
            // 这个方法可以平滑的执行生命周期
            cycleToPath(r, closestPreExecutionState, transaction);
        }
        // 执行item的execute方法
        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
	...
    }
}
```

上节代码我们知道启动Activity时创建的`ClientTransactionItem`为**`LaunchActivityItem`**，后面将执行它的execute方法。所以我们来看一下LaunchActivityItem里面的execute方法的实现

```java
//LaunchActivityItem.java
@Override
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client, mAssistToken);
    // 处理启动Activity前的事项
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```

client是ActivityThread，方法中调用ActivityThread#handleLaunchActivity来启动Activity。

```java
//ActivityThread.java
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ···
    // 初始化WindowManager
    WindowManagerGlobal.initialize();
    ···
    // 启动Activity
    final Activity a = performLaunchActivity(r, customIntent);
    ···
    return a;
}
```

Activity启动的核心代码都在performLaunchActivity方法里。

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ···
    // 创建要启动Activity的上下文环境
    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        // 用类加载器创建该Activity实例，也就是通过反射来创建
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess();
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }

    try {
        // 创建Application实例
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);
        ···
        if (activity != null) {
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config = new Configuration(mCompatConfiguration);
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                    + r.activityInfo.name + " with config " + config);
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }
            appContext.setOuterContext(activity);
            // 调用activity.attach进行初始化
            // 建立Context与Activity的联系，Window将会在这里创建
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.configCallback,
                    r.assistToken);

            ···
            // 回调onCreate
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            r.activity = activity;
        }
        // 更新LifeCycle状态
        r.setState(ON_CREATE);
        ···
    return activity;
}
```

至此Activity就已经被成功创建，并执行了其生命周期相关方法。

以上整一个根Activity的启动过程了。

#### 总结

对应上面的第一张图，根Activity的启动一共涉及四个进程。Launcher进程，SystemServer进程，Zygote进程，和应用程序进程。

首先Launcher进程会向ATMS请求创建Activity，ATMS会判断根Activity所需的应用程序进程是否存在并启动。如果不存在就会请求Zygote进程创建应用程序进程。应用程序进程启动后，ATMS就会请求应用程序进程创建根Activity并启动。其中步骤1是通过Binder进行跨进程通信的，步骤2是通过Scoket进行通信的，步骤4是进行Binder通信的。

![image-20220113120442747](%E6%A0%B9Activity%E7%9A%84%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.assets/image-20220113120442747.png)