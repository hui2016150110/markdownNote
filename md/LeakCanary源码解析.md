### LeakCanary源码解析

#### 内存泄露

今天来讲解一下老生常谈的问题了，内存泄露以及讲解`LeakCanary`是如果检测内存泄露的。

大家都在讲内存泄露，那么内存泄露的最根本的原因是什么？**最根本的原因就是该回收的对象没有被即使回收掉，导致了内存泄露。**要理解这句话，就要对java的垃圾回收机制有一定的了解了。什么是垃圾回收呢？就是java虚拟机在运行的时候会触发垃圾回收的机制，将那些没有用的，占用内存的对象回收掉。java虚拟机是怎么判断这个对象有没有用呢？是根据GC ROOT的可达性算法去判断的。就是如果一个对象，通过GC ROOT可达，那么它就是有用的对象。否则他就是没用的对象。

#### 引用

无论是哪种方法，判断对象的存活都与“引用”有关。那么现在虚拟机对引用的期望是：当内存空间足够的时候，能保存在内存中；如果内存空间在进行垃圾回收之后还是非常紧张，则可以抛弃这些对象。所以就产生了四种引用

1.强引用：通过关键字new出来的对象，只要强引用还在，那么垃圾回收器就不会回收被引用的对象。

2.软引用：在系统将要发生内存溢出异常之前，将会把这些对象列入回收范围之中进行第二次回收。

3.弱引用：被弱引用引用的对象只能存活到下一垃圾回收发生之前。

4.虚引用：一个对象有无虚引用的存在，完全不影响其生存时间。也无法通过虚引用获取一个对象实例，为对象设置虚引用的目的就是能在这个对象被回收的时候，收到一个系统通知。

结合以上两点，可以得出，**如果一个对象是被强引用，那么垃圾回收是不会回收这一部分对象的。那么同时如果虚拟机判断了GC ROOT不可达，那么又回收不了这个对象，那么就可以说这个对象造成了内存泄露**。因为没有地方使用到这个对象了，而这个对象却还占用着的内存，所以他导致了内存的浪费，就是我们所说的内存泄露。

#### 弱引用与引用队列

```kotlin
fun main() {
    val referenceQueue = ReferenceQueue<Pair<String, Int>?>()
    var pair: Pair<String, Int>? = Pair("小黄", 24)
    val weakReference = WeakReference(pair, referenceQueue)

    println(referenceQueue.poll()) //null

    pair = null

    System.gc()
    //GC 后休眠一段时间，等待 pair 被回收
    Thread.sleep(4000)

    println(referenceQueue.poll()) //java.lang.ref.WeakReference@d716361
}
```

可以看到，在 GC 过后 `referenceQueue.poll()` 的返回值变成了**非 null**，这是由于 `WeakReference` 和 `ReferenceQueue` 的一个组合特性导致的：**在声明一个 `WeakReference` 对象时如果同时传入了 `ReferenceQueue` 作为构造参数的话，那么当 `WeakReference` 持有的对象被 GC 回收时，JVM 就会把这个弱引用存入与之关联的引用队列之中。依靠这个特性，我们就可以实现内存泄露的检测了**

例如，当用户按返回键退出 `Activity` 时，正常情况下该 `Activity` 对象应该在不久后就被系统回收，我们可以监听 `Activity` 的 `onDestroy` 回调，在回调时把 `Activity` 对象保存到和 `ReferenceQueue` 关联的 `WeakReference` 中，在一段时间后（可以主动触发几次 GC）检测 `ReferenceQueue` 中是否有值，如果一直为 null 的话就说明发生了内存泄露。`LeakCanary` 就是通过这种方法来实现的。

#### LeakCanary源码分析

讲解了上面的一些前置知识，有助于大家更好的理解`LeakCanary`的工作原理。接下来就是讲解`LeakCanary`的工作流程和具体的细节了。这篇博客是基于2.8.1版本讲解的。

LeakCanary 2.8.1 版本中，LeakCanary 将初始过程交由 `MainProcessAppWatcherInstaller` 这个 `ContentProvider` 来自动完成

##### MainProcessAppWatcherInstaller#onCreate

```kotlin
//这是一个ContentProvider
internal class MainProcessAppWatcherInstaller : ContentProvider() {
    
  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }

}
```

由于 `ContentProvider` 会在 `Application` 被创建之前就由系统调用其 `onCreate()` 方法来完成初始化，所以 LeakCanary 通过 `AppWatcherInstaller` 就可以拿到 `Context` 来完成初始化并随应用一起启动，通过这种方式就简化了使用者的引入成本。

`MainProcessAppWatcherInstaller` 最终会将 `Application` 对象传给 `AppWatcher` 的 `manualInstall(Application)` 方法

##### AppWatcher#manualInstall

```kotlin
@JvmOverloads
fun manualInstall(
    application: Application,
    retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),//这里也是一个默认的参数，5秒
    watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)//这里有默认的参数
) {
    checkMainThread()
    if (isInstalled) {
        throw IllegalStateException(
            "AppWatcher already installed, see exception cause for prior install call", installCause
        )
    }
    check(retainedDelayMillis >= 0) {
        "retainedDelayMillis $retainedDelayMillis must be at least 0 ms"
    }
    installCause = RuntimeException("manualInstall() first called here")
    this.retainedDelayMillis = retainedDelayMillis
    if (application.isDebuggableBuild) {
        LogcatSharkLog.install()
    }
   //初始化一些配置，InternalLeakCanary类invoke方法，配置泄漏Listener，GC触发器，Dump类。
    LeakCanaryDelegate.loadLeakCanary(application)

    //调用各种install，其实就是为各种InstallableWatcher注册一些监听事件，这里有四个InstallableWatcher
    watchersToInstall.forEach {
        it.install()
    }
}
```

##### AppWatcher#appDefaultWatchers

```kotlin
fun appDefaultWatchers(
  application: Application,
  reachabilityWatcher: ReachabilityWatcher = objectWatcher
): List<InstallableWatcher> {
  return listOf(
    ActivityWatcher(application, reachabilityWatcher),
    FragmentAndViewModelWatcher(application, reachabilityWatcher),
    RootViewWatcher(reachabilityWatcher),
    ServiceWatcher(reachabilityWatcher)
  )
}
```

可以看出这里面有四中类型的`InstallableWatcher`。里面的实现大同小异

##### ActivityWatcher

```kotlin
class ActivityWatcher(
  private val application: Application,
  private val reachabilityWatcher: ReachabilityWatcher
) : InstallableWatcher {

  private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }

  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }

  override fun uninstall() {
    application.unregisterActivityLifecycleCallbacks(lifecycleCallbacks)
  }
}
```

看一下`ActivityWatcher`就可以了，调用install方法，其实就是为了注册监听而已，看一下这个`lifecycleCallbacks`，可以看出，当`Acrivity`发生`onDestory`的时候，会回调到`onActivityDestroyed`这里面来。而这里面调用的是 `reachabilityWatcher.expectWeaklyReachable`。所以去看一下这个函数大概就知道`LeakCanary`到底是怎么工作的了。这之前，看一下`reachabilityWatcher`是什么东西，看会`appDefaultWatchers`函数可以知道，`appDefaultWatchers`其实就是`objectWatcher`。

##### objectWatcher的初始化

```kotlin
val objectWatcher = ObjectWatcher(
    clock = { SystemClock.uptimeMillis() },
    checkRetainedExecutor = {
        check(isInstalled) {
            "AppWatcher not installed"
        }
        mainHandler.postDelayed(it, retainedDelayMillis)
    },
    isEnabled = { true }
)
```

所以`expectWeaklyReachable`其实就是`ObjectWatcher`的`expectWeaklyReachable`函数

##### ObjectWatcher#expectWeaklyReachable

这个函数我们需要一步一步慢慢看

```kotlin
  @Synchronized override fun expectWeaklyReachable(
    watchedObject: Any,
    description: String
  ) {
    if (!isEnabled()) {
      return
    }
    //首先去移除那些没有发生泄露的对象，第一次先忽略
    removeWeaklyReachableObjects()
    //生成一个key
    val key = UUID.randomUUID()
      .toString()
    val watchUptimeMillis = clock.uptimeMillis()
   //通过这个key，构建一个弱引用对象，同时传入一个引用队列
    val reference =
      KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    SharkLog.d {
      "Watching " +
        (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
        (if (description.isNotEmpty()) " ($description)" else "") +
        " with key $key"
    }
	//然后将这个弱引用对象存入watchedObjects
    watchedObjects[key] = reference
    checkRetainedExecutor.execute {
      moveToRetained(key)//这个方法会延迟5秒执行
    }
  }
```

##### ObjectWatcher#moveToRetained

```kotlin
@Synchronized private fun moveToRetained(key: String) {
    //移除那些没有发生泄露的对象
    removeWeaklyReachableObjects()
    //执行了移除操作后，在watchedObjects中还能找到对饮的弱引用的话，说明这个对象发生了泄露了
    val retainedRef = watchedObjects[key]
    if (retainedRef != null) {
        retainedRef.retainedUptimeMillis = clock.uptimeMillis()///记录当前时间
        onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
}
```

##### ObjectWatcher#removeWeaklyReachableObjects

```kotlin
private fun removeWeaklyReachableObjects() {
   //根据弱引用与引用队列那一节的分析，我们可以知道如果能在queue中的对象都是没有泄露的对象。这里的操作是遍历queue中的对象，然后根据它的key，将key对应的watchedObjects里面的对象移除。如果执行了该操作之后watchedObjects中还存在对应的key，那么说明发生GC的时候，这个对象没有被放入引用队列，也就是说该对象没有被回收，所以可以确定该对象造成了内存泄露
    var ref: KeyedWeakReference?
    do {
        ref = queue.poll() as KeyedWeakReference?
        if (ref != null) {
            watchedObjects.remove(ref.key)
        }
    } while (ref != null)
}
```

`moveToRetained` 方法就用于判断指定 key 关联的对象是否已经泄露，如果没有泄露则移除对该对象的弱引用，有泄露的话则更新其 `retainedUptimeMillis` 值，以此来标记其发生了泄露，并同时通过回调 `onObjectRetained` 来分析内存泄露链。

以上就是LeakCanary的关于如何检测内存泄露的源码分析了。其实检测内存泄露这一部分代码确实也比较简单，难的是获取内存泄露链和Dump那一部分。不过这两部分不在今天的讨论范围了，后续有机会的话，可以单独拿出来讲一下。好了今天的博客就这么多了，喜欢的小伙伴点个关注和赞吧。