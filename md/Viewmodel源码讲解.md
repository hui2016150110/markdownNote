### Viewmodel源码解析

现在Viewmodel相信大家都比较熟悉了，adnroid的官网上也介绍了它的一些特性。具体的这里就不多讲了。今天就来讲解它的源码，讲解为什么它能够在配置发生变化的时候还能保存数据。

现在在Activity和Fragment中怎么去初始化一个viewmodel呢？一般来讲就是通过谷歌给我们提供的扩展函数去初始化，具体如下

```kotlin
val plantingsViewModel:PlantListViewModel by viewModels()
```

所以今天，我就从这一个函数开始入手去看看Viewmodel内部是怎么实现的。

#### ActivityViewModelLazy.kt

```kotlin
//ActivityViewModelLazy.kt
@MainThread
public inline fun <reified VM : ViewModel> ComponentActivity.viewModels(
    noinline factoryProducer: (() -> Factory)? = null
): Lazy<VM> {
    val factoryPromise = factoryProducer ?: {
        defaultViewModelProviderFactory //ViewModelProvider.Factory类型
    }
	//viewModelStore是ViewModelStore类型
    return ViewModelLazy(VM::class, { viewModelStore }, factoryPromise)
}
```

#### ViewModelProvider#ViewModelLazy

```kotlin
public class ViewModelLazy<VM : ViewModel> (
    private val viewModelClass: KClass<VM>,
    private val storeProducer: () -> ViewModelStore,
    private val factoryProducer: () -> ViewModelProvider.Factory
) : Lazy<VM> {
    private var cached: VM? = null

    override val value: VM
    	//获取ViewModel
        get() {
            //将cache赋值给Viewmodel
            val viewModel = cached
            //如果viewmodel为空，那么就通过 ViewModelProvider.get方法返回viewmodel，并将cache赋值
            return if (viewModel == null) {
                val factory = factoryProducer()
                val store = storeProducer()
                ViewModelProvider(store, factory).get(viewModelClass.java).also {
                    cached = it
                }
            } else {
                //如果有viewmodel不为空，那么直接放回
                viewModel
            }
        }

    override fun isInitialized(): Boolean = cached != null
}
```

从这里的源码可以看出，这里面涉及的类有以下四个类

`ViewModel`：viewmodel本身

`ViewModelStore`：就是用来保存`ViewModel`的

 `ViewModelProvider.Factory`：Factory 接口的实现负责实例化 `ViewModel`。

`ViewModelProvider`：为外部提供 创建`ViewModel` 的方法。

看来上面的代码流程，我们知道下一步的计划就是去看ViewModelProvider.get方法

#### ViewModelProvider

```java
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}
//真正获取viewmodel的地方
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    //首先会通过key去ViewModelStore里面去看有没有viewmodel，这个key其实也就是你的viewModelClass相关的一些名字的拼接
    ViewModel viewModel = mViewModelStore.get(key);

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
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
    } else {
        //当缓存中没有时，就调用Factory的create方法创建ViewModel对象
        viewModel = mFactory.create(modelClass);
    }
    //将ViewModel放入ViewModelStore缓存
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}
```

看到这是不是有点缓存内味，先从缓存里拿数据，如果有数据就返回，如果缓存里没有就重新创建数据，并且将数据put到缓存中。所以要了解`ViewModelStore`

中取出来的`viewmodel`为什么不变，那么就需要去看看`ViewModelStore`是怎么创建的了。

这时候就需要回到`getViewModelStore()`函数了

#### ComponentActivity#getViewModelStore

```java
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                                        + "Application instance. You can't request ViewModel before onCreate call.");
    }
    ensureViewModelStore();
    return mViewModelStore;
}

void ensureViewModelStore() {
    if (mViewModelStore == null) {
        //可以看到一开始在mViewModelStore等于null时，则先获取了NonConfigurationInstances对象
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
}

static final class NonConfigurationInstances {
    Object custom;
    ViewModelStore viewModelStore;
}
```

`NonConfigurationInstances`是用来存储`ViewModelStore`的一个类。当`NonConfigurationInstances`对象不为null时，先直接从`NonConfigurationInstances`拿`ViewModelStore`，如果拿不到则就直接`new `一个`ViewModelStore`对象。这就是获取`ViewModelStore`的过程。

这里有一个**NonConfigurationInstances** ，其对象是由调用`getLastNonConfigurationInstance()`获取，而`getLastNonConfigurationInstance`所返回的实例是由在`onRetainNonConfigurationInstance`中保存的。

#### ComponentActivity#onRetainNonConfigurationInstance

```java
//Retain all appropriate non-config state. You can NOT override this yourself! Use a androidx.lifecycle.ViewModel if you want to retain your own non config state.
//保留所有适当的非配置状态。你不能自己覆盖它！如果您想保留自己的非配置状态，请使用 androidx.lifecycle.ViewModel。
public final Object onRetainNonConfigurationInstance() {
    // Maintain backward compatibility.
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
	//将viewModelStore存储到NonConfigurationInstances中
    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    nci.viewModelStore = viewModelStore;
    return nci;
}
```

**`onRetainNonConfigurationInstance`**是在一些配置发生改变的时候调用的，比如横竖屏切换的时候。这样一来在屏幕切换前，先把`viewModelStore`存储起来，然后屏幕切换后，再把`viewModelStore`拿出来，这样，这个`viewModelStore`相当于没有发生变化，从而里面的viewmodel也得以保存。

以上就是为什么viewmodel能在屏幕发生旋转的时候，里面的viewmodel能保存的原因了。

那么总结一下ViewModel能保存数据的原因：

**首先，第一次进入Activity时，由于ViewModel什么的都没有初始化，这时候，会通过ViewModelProvider相关的类来创建viewmodel，并且将这个viewmodel保存在ViewModelStore里面的HashMap中**

**当手机配置发生改变之后，会触发onRetainNonConfigurationInstance，对ViewModelStore进行存储；**

**Activity重建后，Activity会重新走onCreate生命周期，并且会再次去获取ViewModel对象。**

**但是这次ViewModel的获取与第一次创建不同，它会通过ViewModelStoreOwner先获取该Activity重建之前所保存的ViewModelStore，接着在ViewModelStore中根据Key，找到重建之前的ViewModel，使得这两次viewmodel数据一样**

