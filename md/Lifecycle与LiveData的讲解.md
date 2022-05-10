### `Lifecycle`与`LiveData`的讲解

#### lifecycle

`Lifecycle`是生命周期感知型组件，什么是生命感知型组件？就是与`Activity`或者`Fragment`绑定之后，可执行一些操作来响应Activity和Fragment的生命周期状态的变化。

`lifecycle`是一个类，用于存储有关组件（如 `Activity` 或 `Fragment`）的生命周期状态的信息，并且允许其他对象观察此状态。`Lifecycle`使用两种主要枚举跟踪其关联组件的生命周期状态：这两个枚举类分别是State和Event。

**State**：当前生命周期所处状态。有以下状态：`CREATED`，`STARTED`，`RESUMED`，`DESTROYED`，`INITIALIZED`

**Event**：当前生命周期改变对应的事件。    有以下事件：`ON_CREATE`,`ON_START`,`ON_RESUME`,`ON_PAUSE`,`ON_STOP`,`ON_DESTROY`,`ON_ANY`;

![生命周期状态示意图](https://developer.android.com/images/topic/libraries/architecture/lifecycle-states.svg?hl=zh-cn)

注意这张事件和状态的转换图，它的意思是，从一开始的`INITIALIZED`状态，经过了`onCreate`事件，也就是`Activity`或者`Fragment`的`onCreate`函数之后，他就变成了`CREATED`状态，其他的事件和状态的转换同理。就不一一列举了。



**`LifecycleObserver`接口（ `Lifecycle`观察者）**：实现该接口的类，通过注解的方式，可以通过被`LifecycleOwner`类的`addObserver(o:LifecycleObserver)`方法注册，被注册后，`LifecycleObserver`便可以观察到`LifecycleOwner`的生命周期事件。

**`LifecycleOwner`接口（`Lifecycle`持有者）**：实现该接口的类持有生命周期(`Lifecycle`对象)，`Lifecycle`对象的改变会被其注册的观察者`LifecycleObserver`观察到并触发其对应的事件。

通过以上介绍可以知道：实现 `LifecycleObserver`的组件可与实现 `LifecycleOwner`的组件完美配合，因为`LifecycleOwner`可以提供生命周期，而`LifecycleObserver`可以注册以便观察组件的生命周期。

`LifeCycle`的用法：以官网的定位功能为例

```kotlin
//没有使用LifeCycle的情况
class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this) { location ->
            // update UI
        }
    }

    public override fun onStart() {
        super.onStart()
        Util.checkUserStatus { result ->
            // 如果在活动停止后调用这个回调会怎样？可能会崩溃
            if (result) {
                myLocationListener.start()
            }
        }
    }

    public override fun onStop() {
        super.onStop()
        myLocationListener.stop()
    }

}
```

```kotlin
//使用lifeCycle的方式
class MyActivity : AppCompatActivity() {
    private lateinit var myLocationListener: MyLocationListener

    override fun onCreate(...) {
        myLocationListener = MyLocationListener(this, lifecycle) { location ->
            // update UI
        }
        Util.checkUserStatus { result ->
            if (result) {
                myLocationListener.enable()
            }
        }
        lifeCycle.addObserver(myLocationListener)

    }
}
```

```kotlin
internal class MyLocationListener(
        private val context: Context,
        private val lifecycle: Lifecycle,
        private val callback: (Location) -> Unit
): LifecycleObserver {

    private var enabled = false

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun start() {
        if (enabled) {
            // connect
        }
    }

    fun enable() {
        enabled = true
        if (lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
            // connect if not connected
        }
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun stop() {
        // disconnect if connected
    }
}
```

来看一下整体的生命周期的调用一个流程，如下图。

[ShymanZhu](https://blog.csdn.net/sd_zhuzhipeng)![在这里插入图片描述](https://img-blog.csdnimg.cn/b83da6947ef845d18c7d6f75c2c5f33d.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zNzg3Mzc4Ng==,size_16,color_FFFFFF,t_70#pic_center)

LifeCycle的作用是让其他组件也可以检测Activity和Fragment生命周期的变化，然后做出相应的响应，并且将Activity的生命周期函数简化。它更大的作用是和其他组件配合使用，比如LiveData。

#### Livedata

`LiveData`是一种可观察的数据存储器类。与常规的可观察类不同，**`LiveData` 具有生命周期感知能力**，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。**这种感知能力可确保 `LiveData` 仅更新处于活跃生命周期状态的应用组件观察者。**

如果观察者（由 `Observer`类表示）的生命周期处于 **STARTED或 RESUMED**状态，则 `LiveData` 会认为该观察者处于活跃状态。`LiveData` 只会将更新通知给活跃的观察者。为观察 `LiveData`对象而注册的非活跃观察者不会收到更改通知。

那么怎么使用LiveData呢？

1. 创建 `LiveData` 的实例以存储某种类型的数据。这通常在 `ViewModel`类中完成。
2. 创建可定义 `onChanged()`方法的 `Observer`对象，该方法可以控制当 `LiveData` 对象存储的数据更改时会发生什么。通常情况下，您可以在界面控制器（如 Activity 或 Fragment）中创建 `Observer` 对象。
3. 使用 `observe()` 方法将 `Observer` 对象附加到 `LiveData` 对象。`observe()` 方法会采用 `LifecycleOwner` 对象。这样会使 `Observer` 对象订阅 `LiveData` 对象，以使其收到有关更改的通知。通常情况下，您可以在界面控制器（如 Activity 或 Fragment）中附加 `Observer` 对象。这一步一般是在onCreate中去完成。
4. 使用`setValue`或者`postValue`的方法去更新`LiveDate`里面的值，以便观察者能够感知`livedate`发生了变化。

下面是一个例子：
```kotlin
class TestViewModel: ViewModel(){
    // 1、创建LiveData的实例以存储某种类型的数据。通常再viewmodel中完成，userName的值改变后，会通知它的观察者
    val userName: MutableLiveData<String> = MutableLiveData()

}
```

```kotlin
    lateinit var testViewModel: TestViewModel
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        ...
        
        //2、通过livedata的observe()方法，设置LifecycleOwner和Observer对象
        testViewModel.userName.observe(this){
            //3、实现Observer 即userName更改后需要执行的操作
        })
        
        ...
    }
```

当更新存储在 `LiveData` 对象中的值时，它会触发所有已注册的观察者（只要附加的 `LifecycleOwner` 处于活跃状态）。LiveData 允许界面控制器观察者订阅更新。当 `LiveData` 对象存储的数据发生更改时，界面会自动更新以做出响应。

来看一下LiveData的源码，它的源码也非常简单。我们直接从setValue开始看

```java
   // 首先第一步，通过setValue，将value的值赋值给mData，然后调用dispatchingValue

	@MainThread
    protected void setValue(T value) {
        assertMainThread("setValue");
        mVersion++;
        mData = value;
        dispatchingValue(null);
    }
```

```java
    void dispatchingValue(@Nullable ObserverWrapper initiator) {
        if (mDispatchingValue) {
            mDispatchInvalidated = true;
            return;
        }
        mDispatchingValue = true;
        do {
            mDispatchInvalidated = false;
            if (initiator != null) {
                considerNotify(initiator);
                initiator = null;
            } else {
                //第二部，到了dispatchingValue里面之后，然后循环遍历，开始调用considerNotify
                for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                        mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
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

```java
// 最后一步，调用considerNotify，判断观察者是不是处于一个活跃状态，如果不是，直接返回，也就是如果这个观察者不是活跃状态，那么将收不到livedata这次改变的通知，如果处于观察者，那么就调用观察者的onChanged函数    
private void considerNotify(ObserverWrapper observer) {
        if (!observer.mActive) {
            return;
        }
        // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
        //
        // we still first check observer.active to keep it as the entrance for events. So even if
        // the observer moved to an active state, if we've not received that event, we better not
        // notify for a more predictable notification order.
        if (!observer.shouldBeActive()) {
            observer.activeStateChanged(false);
            return;
        }
        if (observer.mLastVersion >= mVersion) {
            return;
        }
        observer.mLastVersion = mVersion;
        observer.mObserver.onChanged((T) mData);
    }
```

以上就是LiveData的讲解，这一篇讲解比较基础，以为lifecycle与livedate无论原来还是使用比较简单的，他们配合起来使用确实也比较方便。原理无非就是使用了观察者模式而已。