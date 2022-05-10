### Android中的各种Context

#### Context家族之间的关系

![image-20220117180456864](Android%E4%B8%AD%E7%9A%84%E5%90%84%E7%A7%8DContext%E5%AF%B9%E6%AF%94.assets/image-20220117180456864.png)

看以上这幅图，我们知道各个Context之间的关系。

&ensp;&ensp;首先`Context`是一个抽象类，它的实现由两个一个是`ContextImpl`，它是真是实现了`Context`里面的各种方法。`ContextWrapper`里面持有一个`ContextImpl`变量——mBase，调用`ContextWrapper`的实现方法，最终都是通过mBase去调用`ContextImpl`的实现方法。这里用的设计模式是装饰模式，`ContextWrapper`是装饰类。`ContextThemeWrapper`，`Service`，`Application`都是`ContextWrapper`的子类，它们都可以通过mBase去调用`Context`的方法，同样它们也是装饰类，在`contextWrapper`的基础上添加了不同的功能。`ContextThemeWrapper` 中包含和主题相关的方怯（比如 `getTheme` 方法），因此，需要主题的 `Activity` 继承 `ContextThemeWrapper` 。知道了Context之间的类的关系，着仅仅是了解Context的第一步，要深入了解Context，我们就必须知道Context是做什么的，它有什么作用。那么Context是做什么的呢，或者说Context有什么作用呢？

#### Context是用来做什么的？

要了解这个问题，那么就必须去看代码了，上面我们说了Context是一个抽象类，所以去看一下Context的抽象方法不就能大概了解到Context是做什么的了吗？这里呢不会列举所有的方法和变量，只会列举一些常见的方法

```java
/**
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
//翻译过来就是，Context是有关应用程序环境的全局信息的接口。它的实现呢，由Android系统提供。并且它能允许访问特定于应用程序的资源和类，以及调用应用程序级操作，例如启动活动、广播和接收意图等。
public abstract class Context {

    //开启活动相关的，还有startActivityForResult等
    public abstract void startActivity(@RequiresPermission Intent intent,
            @Nullable Bundle options);
    //与广播相关的，还有其他的方法
    public abstract void sendBroadcast(Intent intent,
                                       @Nullable String receiverPermission,
                                       @Nullable Bundle options);
    public abstract Intent registerReceiver(@Nullable BroadcastReceiver receiver,
                                            IntentFilter filter);
    public abstract void unregisterReceiver(BroadcastReceiver receiver);
    
    // 与服务相关的方法
    public abstract ComponentName startService(Intent service);
    public abstract boolean stopService(Intent service);
    public abstract boolean bindService(@RequiresPermission Intent service,
            @NonNull ServiceConnection conn, @BindServiceFlags int flags);
    public abstract void unbindService(@NonNull ServiceConnection conn);

    //与内容提供者相关的方法
    public abstract ContentResolver getContentResolver();
    
    //与获取资源相关的方法
    public abstract AssetManager getAssets();
    public abstract Resources getResources();
    public abstract PackageManager getPackageManager();
 	public final CharSequence getText(@StringRes int resId) {
        return getResources().getText(resId);
    }
    public final String getString(@StringRes int resId) {
        return getResources().getString(resId);
    }
    public final int getColor(@ColorRes int id) {
        return getResources().getColor(id, getTheme());
    }
    public final Drawable getDrawable(@DrawableRes int id) {
        return getResources().getDrawable(id, getTheme());
    }
    public final ColorStateList getColorStateList(@ColorRes int id) {
        return getResources().getColorStateList(id, getTheme());
    }
    ...
        
     // 应用相关信息方法
    public abstract ApplicationInfo getApplicationInfo();
    public abstract String getPackageName();
    public abstract Looper getMainLooper();
    public abstract int checkPermission(@NonNull String permission, int pid, int uid);
    
    // 文件相关方法
    public abstract File getSharedPreferencesPath(String name);
    public abstract File getDataDir();
    public abstract boolean deleteFile(String name);
    public abstract File getExternalFilesDir(@Nullable String type);
    public abstract File getCacheDir();
    ...
    public abstract SharedPreferences getSharedPreferences(String name, @PreferencesMode int mode);
    public abstract boolean deleteSharedPreferences(String name);
    
    // 与数据库相关
    public abstract SQLiteDatabase openOrCreateDatabase(...);
    public abstract boolean deleteDatabase(String name);
    public abstract File getDatabasePath(String name);
    ...
    
    // 其它
    public void registerComponentCallbacks(ComponentCallbacks callback) { ... }
    public void unregisterComponentCallbacks(ComponentCallbacks callback) { ... }
    
}
```

综合以上的方法可以看出，其实Context是一个集众多功能于一身的一个类，有自己实现的方法也有他子类实现的方法。它主要负责以下几件事

+ **负责与四大组件交互，例如启动Activity，发送广播等**
+ **提供获取应用内各种资源的方法**
+ **提供与持久性存储相关的功能，比如数据库，文件，sp等**
+ **还有一些其他的辅助功能，比如生命周期**

#### 各种Context的区别

##### ContextImpl

```java
//Context API 的通用实现，它为 Activity 和其他应用程序组件提供了基本的上下文对象。
class ContextImpl extends Context{

    // 静态函数，这里用于创建了Application的context和Service的Context
    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo,
            String opPackageName) {
        ...
    }

     // 静态函数，这里用于创建了Activity的context
    @UnsupportedAppUsage
    static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
            Configuration overrideConfiguration) {
       ...
    }

}
```

这里提供了创造各种context的函数，然后还有Context里面各种函数的实现。

##### ContextWrapper

```java
public class ContextWrapper extends Context {
  Context mBase;
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
    public Context getBaseContext() {
        return mBase;
    }
    ...
}
```

​	这个类比较简单，主要讲解它的mBase成员变量，它的所有功能最终都是通过这个mBase变量去实现的，这个mBase变量可以看出是一个`Context`类型，它是通过`attachBaseContext`传过来的，这个方法的调用时机是在创建Context之后调用的，比如如果我们要创建一个`Application`的话，那么就首先会创建`ContextImpl`，然后调用`Application`的`attachBaseContext`，将`ContextImpl`传入。mBase最终的类型是`ComtextImpl`，所以`ContextWrapper`其实只是一个装饰类，他最终的功能还是调用它的mBase去实现的。

##### ContextThemeWrapper

```java
/**
 * 一个上下文包装器，允许您修改或替换被包装的上下文的主题。
 */
public class ContextThemeWrapper extends ContextWrapper {
    @UnsupportedAppUsage
    private int mThemeResource;
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P, trackingBug = 123768723)
    private Resources.Theme mTheme;
    @UnsupportedAppUsage
    private LayoutInflater mInflater;
    private Configuration mOverrideConfiguration;
    @UnsupportedAppUsage
    private Resources mResources;
	...

    /**
    *调用以在此上下文上设置“覆盖配置”——这是一个响应应用到上下文的标准配置的一个或多个值的配置。有关详细信息，请参阅 Context.createConfigurationContext(Configuration)。此方法只能调用一次，并且必须在调用 getResources() 或 getAssets() 之前调用。
    */
    public void applyOverrideConfiguration(Configuration overrideConfiguration) {
        if (mResources != null) {
            throw new IllegalStateException(
                    "getResources() or getAssets() has already been called");
        }
        if (mOverrideConfiguration != null) {
            throw new IllegalStateException("Override configuration has already been set");
        }
        mOverrideConfiguration = new Configuration(overrideConfiguration);
    }

    public Configuration getOverrideConfiguration() {
        return mOverrideConfiguration;
    }

    @Override
    public AssetManager getAssets() {
        // Ensure we're returning assets with the correct configuration.
        return getResourcesInternal().getAssets();
    }

    //没有直接调用mBase的函数
    @Override
    public Resources getResources() {
        return getResourcesInternal();
    }

    private Resources getResourcesInternal() {
        if (mResources == null) {
            if (mOverrideConfiguration == null) {
                mResources = super.getResources();
            } else {
                final Context resContext = createConfigurationContext(mOverrideConfiguration);
                mResources = resContext.getResources();
            }
        }
        return mResources;
    }

    @Override
    public void setTheme(int resid) {
        if (mThemeResource != resid) {
            mThemeResource = resid;
            initializeTheme();
        }
    }

    public void setTheme(@Nullable Resources.Theme theme) {
        mTheme = theme;
    }

    /** @hide */
    @Override
    @UnsupportedAppUsage
    public int getThemeResId() {
        return mThemeResource;
    }
    
	//没有直接调用mBase的函数
    @Override
    public Resources.Theme getTheme() {
        if (mTheme != null) {
            return mTheme;
        }

        mThemeResource = Resources.selectDefaultTheme(mThemeResource,
                getApplicationInfo().targetSdkVersion);
        initializeTheme();
        return mTheme;
    }

    @Override
    public Object getSystemService(String name) {
        if (LAYOUT_INFLATER_SERVICE.equals(name)) {
            if (mInflater == null) {
                mInflater = LayoutInflater.from(getBaseContext()).cloneInContext(this);
            }
            return mInflater;
        }
        return getBaseContext().getSystemService(name);
    }

    protected void onApplyThemeResource(Resources.Theme theme, int resId, boolean first) {
        theme.applyStyle(resId, true);
    }

    @UnsupportedAppUsage
    private void initializeTheme() {
        final boolean first = mTheme == null;
        if (first) {
            mTheme = getResources().newTheme();
            final Resources.Theme theme = getBaseContext().getTheme();
            if (theme != null) {
                mTheme.setTo(theme);
            }
        }
        onApplyThemeResource(mTheme, mThemeResource, first);
    }
}

```

​	这个类的代码其实不多，其主要有一个`Configuration`类型的成员变量，还有注意与主题，资源相关的函数，主题资源相关的函数是没有直接调用mBase的方法的，**这说明了`ContextThemeWrapper`与`ContextImpl`与资源，主题相关的行为是不一样的**。

Application，Service，Activity相关的Context都是装饰器，都在他们父类的基础上增加了一些其他功能，这里因我们经常使用到这三个类，就不一一介绍了，相关的知识点，大家可以通过他们的源码去熟悉去了解。

`ContextImpl`，`ContextWrapper`，`ContextThemeWrapper`三者的区别

+ `ContextWrapper`、`ContextThemeWrapper` 都是 `Context` 的装饰类，二者的区别在于 `ContextThemeWrapper` 有自己的 `Theme` 以及 `Resource`，它的行为是自己实现的，而`ContextWrapper`没有自己的行为，都是通过调用mBase去实现的。
+ `ContextImpl` 是 `Context` 的主要实现类，**`Activity`、`Service` 和 `Application` 的mBase都是由它创建的。**

#### mBase的创建过程

##### Activity的mBase创建过程

上面讲到了几种Context的区别，但是他们中最重要的mBase是怎么来的，还没有详细讲解，所以接下来讲解一下Application（Service与Application一样）与Activity的mBase是怎么来的。

首先看Activity。我们上一篇文章讲解了[根Activity的启动](https://blog.csdn.net/weixin_37873786/article/details/122480166),Activity启动的核心代码都在performLaunchActivity方法里，现在再去回顾一下这一部分的内容。

```java
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        ContextImpl appContext = createBaseContextForActivity(r);//注释1
        Activity activity = null;
        try {
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
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);//注释2

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

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
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);//注释3

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
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
            r.setState(ON_CREATE);

            // updatePendingActivityConfiguration() reads from mActivities to update
            // ActivityClientRecord which runs in a different thread. Protect modifications to
            // mActivities to avoid race.
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }

```

注释1处，通过`createBaseContextForActivity`创建了`ContextImpl`类型的`appContext` ，注释3处将appContext传入了Activity的attach函数，他最终会调用到`ContextWrapper`的`attachBaseContext`，这样Activity的mBase就创建完成了。

```java
protected void attachBaseContext(Context base) {
    if (mBase != null) {
        throw new IllegalStateException("Base context already set");
    }
    mBase = base;
}
```

##### Application的mBase创建过程

同样是上面的performLaunchActivity的方法，看到注释2处，这里创建了Application，那进去看看创建了Application之后做了什么

```java
public Application makeApplication(boolean forceDefaultAppClass,
                                   Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "makeApplication");

    Application app = null;

    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }

    try {
        java.lang.ClassLoader cl = getClassLoader();
        if (!mPackageName.equals("android")) {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                             "initializeJavaContextClassLoader");
            initializeJavaContextClassLoader();
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        }
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);//注释1
        app = mActivityThread.mInstrumentation.newApplication(
            cl, appClass, appContext);//注释2
        appContext.setOuterContext(app);//注释3
    } catch (Exception e) {
        if (!mActivityThread.mInstrumentation.onException(app, e)) {
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            throw new RuntimeException(
                "Unable to instantiate application " + appClass
                + ": " + e.toString(), e);
        }
    }
    mActivityThread.mAllApplications.add(app);
    mApplication = app;//注释4

    if (instrumentation != null) {
        try {
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            if (!instrumentation.onException(app, e)) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw new RuntimeException(
                    "Unable to create application " + app.getClass().getName()
                    + ": " + e.toString(), e);
            }
        }
    }

    // Rewrite the R 'constants' for all library apks.
    SparseArray<String> packageIdentifiers = getAssets().getAssignedPackageIdentifiers();
    final int N = packageIdentifiers.size();
    for (int i = 0; i < N; i++) {
        final int id = packageIdentifiers.keyAt(i);
        if (id == 0x01 || id == 0x7f) {
            continue;
        }

        rewriteRValues(getClassLoader(), packageIdentifiers.valueAt(i), id);
    }

    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

    return app;
}
```

​	在注释1处通过 `Contextlmpl`的`createAppContext` 越来创建 `Contextlmpl` 。注释2处的代码用来创建 `Application` ，在 `Instrumentation` 的`newApplication` 方法中传入了 `ClassLoader` 类型的对象以及注释1处创建的 `Contextlmpl` 。在注释3处将 `Application` 贼值给 `Contextlmpl` Context 类型的成员变 `mOuterContext` ，这 `Contextlmpl` 中也包含了 `Application` 的引用。在注释4处将 `Application` 赋值给 `LoadedApk` 的成员变 `mApplication` ，这个 `mApplication`是`Application` 类型的对象，它用来代表 `Application` 的`Context` 。下面来查看注释2处的 `Application`是如何创建的， `Instrumentation`的newApplication 方法如下所示

```java
public Application newApplication(ClassLoader cl, String className, Context context)
    throws InstantiationException, IllegalAccessException, 
ClassNotFoundException {
    Application app = getFactory(context.getPackageName())
        .instantiateApplication(cl, className);
    app.attach(context);
    return app;
}
```

创建了Application，然后调用它的attach，attach最终也是调用`ContextWrapper`的`attachBaseContext`方法。这样AppliCation的mBase就创建完成了。



以上就是关于Context家族的知识点了，大家有什么问题欢迎讨论。