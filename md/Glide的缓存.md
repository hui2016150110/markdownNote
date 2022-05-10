### Glide的缓存流程

上一篇讲解了Glide的整体流程，其实很多时候，只有第一次加载图片的时候，我们才会按照那一个流程去走。因为很多时候，我们都是有缓存了。有了缓存之后，加载流程就会稍微变一下了。那么今天，我们就来讲解一下Glide中的缓存。在讲解Glide缓存之后，我建议大家先去了解一下`LinkedHashMap`的实现。因为这里涉及到`LRU`算法。推荐大家一篇关于`LinkedHashMap`的博客：[田小波关于LinkedHashMap的源码分析](https://www.tianxiaobo.com/2018/01/24/LinkedHashMap-%E6%BA%90%E7%A0%81%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90%EF%BC%88JDK1-8%EF%BC%89/)

先来一张Glide缓存的流程图吧，让大家对Glide的流程有一个印象，方便之后的分析，以下流程图是基于配置了允许缓存的流程，配置了不允许缓存的不在本博客的讨论范围。

#### Glide缓存流程图

![image-20220316175513742](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220316175513742.png)

通过上面这个流程图，我们可以知道Glide的缓存可以分为三级，第一个是`ActiveResources`，第二个是`MemoryCache`，第三个是`DiskCache`。后面两个，大家都比较熟悉了，一个是内存缓存，一个是磁盘缓存。

#### ActiveResources

那么就先简单介绍一个`ActiveResources`。先看`ActiveResources`的构造函数，以及它里面的一些成员变量

```java
final class ActiveResources {
    private final boolean isActiveResourceRetentionAllowed;
    private final Executor monitorClearedResourcesExecutor;
    @VisibleForTesting final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
    private final ReferenceQueue<EngineResource<?>> resourceReferenceQueue = new ReferenceQueue<>();

    private ResourceListener listener;

    private volatile boolean isShutdown;
    @Nullable private volatile DequeuedResourceCallback cb;

    ActiveResources(boolean isActiveResourceRetentionAllowed) {
        this(
            isActiveResourceRetentionAllowed,
            java.util.concurrent.Executors.newSingleThreadExecutor(
                new ThreadFactory() {
                    @Override
                    public Thread newThread(@NonNull final Runnable r) {
                        return new Thread(
                            new Runnable() {
                                @Override
                                public void run() {
                                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                                    r.run();
                                }
                            },
                            "glide-active-resources");
                    }
                }));
    }

    //第一个构造方法最终会调这个构造方法
    @VisibleForTesting
    ActiveResources(
        boolean isActiveResourceRetentionAllowed, Executor monitorClearedResourcesExecutor) {
        this.isActiveResourceRetentionAllowed = isActiveResourceRetentionAllowed;
        this.monitorClearedResourcesExecutor = monitorClearedResourcesExecutor;

        monitorClearedResourcesExecutor.execute(
            new Runnable() {
                @Override
                public void run() {
                    //调用该方法
                    cleanReferenceQueue();
                }
            });
    }

    @Synthetic
    void cleanReferenceQueue() {
        //一直循环
        while (!isShutdown) {
            try {
                ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();
                cleanupActiveReference(ref);

                // This section for testing only.
                DequeuedResourceCallback current = cb;
                if (current != null) {
                    current.onResourceDequeued();
                }
                // End for testing only.
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

`ActiveResources`构建完成后，会启动一个后台优先级级别（`THREAD_PRIORITY_BACKGROUND`）的线程，主要就是在while循环里面调用了`resourceReferenceQueue`的`remove`()，这个方法会一直阻塞当前线程，直到有返回值。当`ResourceWeakReference`里面的`EngineResource`被内存回收掉的时候才会有返回值。

看一下`cleanReferenceQueue`方法：

```java
//这个方法在两处被两用，一个就是上面的cleanReferenceQueue中，还有一个就是get方法中
@Synthetic
void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
    synchronized (this) {
        activeEngineResources.remove(ref.key);
		
        if (!ref.isCacheable || ref.resource == null) {
            return;
        }
    }
    //如果是在get()中调用这个方法会走到这里来
		// 回调Engine的onResourceReleased方法
      // 这会导致此资源从active变成memory cache状态
    EngineResource<?> newResource =
        new EngineResource<>(
        ref.resource, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ false, ref.key, listener);
    listener.onResourceReleased(ref.key, newResource);
}
```

接着看一下是如何保存和删除`Resource`的

```java
//保存resource
synchronized void activate(Key key, EngineResource<?> resource) {
    //先将resource封装成ResourceWeakReference
    ResourceWeakReference toPut =
        new ResourceWeakReference(
        key, resource, resourceReferenceQueue, isActiveResourceRetentionAllowed);
	//然后将ResourceWeakReference对象存入队列中
    ResourceWeakReference removed = activeEngineResources.put(key, toPut);
    //如果之前的队列中有相同的key存在的对象，那么应该将之前的对应重置
    if (removed != null) {
        removed.reset();
    }
}

//删除resource。这里代码简单多了，就不过多的分析了
synchronized void deactivate(Key key) {
    ResourceWeakReference removed = activeEngineResources.remove(key);
    if (removed != null) {
        removed.reset();
    }
}
```

这上面要分析的是关于`ResourceWeakReference`

```java
//继承了WeakReference
static final class ResourceWeakReference extends WeakReference<EngineResource<?>> {
    @SuppressWarnings("WeakerAccess")
    @Synthetic
    final Key key;

    @SuppressWarnings("WeakerAccess")
    @Synthetic
    final boolean isCacheable;

    @Nullable
    @SuppressWarnings("WeakerAccess")
    @Synthetic
    Resource<?> resource;

    @Synthetic
    @SuppressWarnings("WeakerAccess")
    ResourceWeakReference(
        @NonNull Key key,
        @NonNull EngineResource<?> referent,
        @NonNull ReferenceQueue<? super EngineResource<?>> queue,
        boolean isActiveResourceRetentionAllowed) {
        //注意这个super函数，这样的作用是，如果referent将要被GC，就会被放入queue中。具体请查阅相关的ReferenceQueue的知识点
        super(referent, queue);
        this.key = Preconditions.checkNotNull(key);
        this.resource =
            referent.isMemoryCacheable() && isActiveResourceRetentionAllowed
            ? Preconditions.checkNotNull(referent.getResource())
            : null;
        isCacheable = referent.isMemoryCacheable();
    }

    void reset() {
        resource = null;
        clear();
    }
}
```

以上大概就是`ActiveResources`的知识点了。`ActiveResource`做为Glide的第一级缓存，保存的是那些活跃的`EngineResource`，即没有被内存回收的数据。同时使用了弱引用，保证了当进行内存回收时能及时回收掉，避免一直占用内存。如果被回收掉就会转移到memory cache中。

预备知识讲解完了，接下来就是进入Glide的缓存流程了。还记得上一篇博客，我们讲解缓存的时候是省略了这一部分。

废话不多说了，直接看Engine.load方法

```java
public <R> LoadStatus load(
    GlideContext glideContext,
    Object model,
    Key signature,
    int width,
    int height,
    Class<?> resourceClass,
    Class<R> transcodeClass,
    Priority priority,
    DiskCacheStrategy diskCacheStrategy,
    Map<Class<?>, Transformation<?>> transformations,
    boolean isTransformationRequired,
    boolean isScaleOnlyOrNoTransform,
    Options options,
    boolean isMemoryCacheable,
    boolean useUnlimitedSourceExecutorPool,
    boolean useAnimationPool,
    boolean onlyRetrieveFromCache,
    ResourceCallback cb,
    Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    //首先先创建EngineKey，大家都清楚，涉及到缓存的，都离不开Key，Glide通过keyFactory.buildKey来创建自己的缓存KEY
    EngineKey key =
        keyFactory.buildKey(
        model,
        signature,
        width,
        height,
        transformations,
        resourceClass,
        transcodeClass,
        options);

    EngineResource<?> memoryResource;
    synchronized (this) {
        //在缓存中去查找是否有一样的key对应的资源，如果有，直接拿出来使用不进行网络请求或者其他操作了
        memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

        //如果没有缓存，那么就需要开始获取资源了
        if (memoryResource == null) {
            return waitForExistingOrStartNewJob(
                glideContext,
                model,
                signature,
                width,
                height,
                resourceClass,
                transcodeClass,
                priority,
                diskCacheStrategy,
                transformations,
                isTransformationRequired,
                isScaleOnlyOrNoTransform,
                options,
                isMemoryCacheable,
                useUnlimitedSourceExecutorPool,
                useAnimationPool,
                onlyRetrieveFromCache,
                cb,
                callbackExecutor,
                key,
                startTime);
        }
    }

    // Avoid calling back while holding the engine lock, doing so makes it easier for callers to
    // deadlock.
    cb.onResourceReady(memoryResource, DataSource.MEMORY_CACHE);
    return null;
}
```

这里面很多关于缓存的代码，首先看`EngineKey`，为什么？因为只有知道了`EngineKey`是什么，我们才能找到我们对应的缓存，所以`EngineKey`至关重要

#### Glide缓存流程分析

##### `EngineKey`的组成

| 组成            | 注释                                                         |
| :-------------- | :----------------------------------------------------------- |
| model           | load的参数                                                   |
| signature       | `BaseRequestOptions`的成员变量，默认会是`EmptySignature.obtain()` 在加载本地resource资源时会变成`ApplicationVersionSignature.obtain(context)` |
| width height    | 如果没有指定`override(int size)`，那么将得到view的size       |
| transformations | 默认会基于`ImageView`的scaleType设置对应的四个`Transformation`； 如果指定了`transform`，那么就基于该值进行设置； 详见`BaseRequestOptions.transform(Transformation, boolean)` |
| resourceClass   | 解码后的资源，如果没有`asBitmap`、`asGif`，一般会是`Object`  |
| transcodeClass  | 最终要转换成的数据类型，根据`as`方法确定，加载本地res或者网络URL，都会调用`asDrawable`，所以为`Drawable` |
| options         | 如果没有设置过`transform`，此处会根据`ImageView`的`scaleType`默认指定一个KV |

那么接下来就是 `memoryResource = loadFromMemory(key, isMemoryCacheable, startTime)`这段代码，这段代码也是核心。首先我们看看`memoryResource` 是什么东西，它是`EngineResource`，也就是说，我们从缓存中找到的对象是`EngineResource`。在阅读缓存之前，我们首先要了解一下什么是`EngineResource`，我们先看一下

##### EngineResource

```java
class EngineResource<Z> implements Resource<Z> {
    private final boolean isMemoryCacheable;
    private final boolean isRecyclable;
    private final Resource<Z> resource;
    private final ResourceListener listener;
    private final Key key;

    private int acquired;
    private boolean isRecycled;

    interface ResourceListener {
        void onResourceReleased(Key key, EngineResource<?> resource);
    }

    EngineResource(
        Resource<Z> toWrap,
        boolean isMemoryCacheable,
        boolean isRecyclable,
        Key key,
        ResourceListener listener) {
        resource = Preconditions.checkNotNull(toWrap);
        this.isMemoryCacheable = isMemoryCacheable;
        this.isRecyclable = isRecyclable;
        this.key = key;
        this.listener = Preconditions.checkNotNull(listener);
    }

    Resource<Z> getResource() {
        return resource;
    }

    boolean isMemoryCacheable() {
        return isMemoryCacheable;
    }

    @NonNull
    @Override
    public Class<Z> getResourceClass() {
        return resource.getResourceClass();
    }

    @NonNull
    @Override
    public Z get() {
        return resource.get();
    }

    @Override
    public int getSize() {
        return resource.getSize();
    }

    @Override
    public synchronized void recycle() {
        if (acquired > 0) {
            throw new IllegalStateException("Cannot recycle a resource while it is still acquired");
        }
        if (isRecycled) {
            throw new IllegalStateException("Cannot recycle a resource that has already been recycled");
        }
        isRecycled = true;
        if (isRecyclable) {
            resource.recycle();
        }
    }


    synchronized void acquire() {
        if (isRecycled) {
            throw new IllegalStateException("Cannot acquire a recycled resource");
        }
        ++acquired;
    }

    @SuppressWarnings("SynchronizeOnNonFinalField")
    void release() {
        boolean release = false;
        synchronized (this) {
            if (acquired <= 0) {
                throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
            }
            if (--acquired == 0) {
                release = true;
            }
        }
        if (release) {
            //这里实际的作用是将ActiveResource缓存移动到内存缓存
            listener.onResourceReleased(key, this);
        }
    }

}
```

我们主要关注两个方法，一个acquire，一个release。acquire方法很简单，就是每调用一次这个方法，就给acquired成员变量加一。release也很简单，每次调用都给acquired成员变量减一，当acquired成员变量为0的时候，调用`listener.onResourceReleased`。这里采用的算法就是垃圾回收里面最简单的**引用计数法**去管理`EngineResource`。接下来我， 要去看`loadFromMemory()`这个方法了

##### Engine#loadFromMemory

```java
private EngineResource<?> loadFromMemory(
    EngineKey key, boolean isMemoryCacheable, long startTime) {
    //如果不允许缓存，直接返回null
    if (!isMemoryCacheable) {
        return null;
    }

    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Loaded resource from active resources", startTime, key);
        }
        return active;
    }

    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
        if (VERBOSE_IS_LOGGABLE) {
            logWithTimeAndKey("Loaded resource from cache", startTime, key);
        }
        return cached;
    }

    return null;
}

//从ActiveResource中去查找，如果找到，这个资源的引用计数就要加一
@Nullable
private EngineResource<?> loadFromActiveResources(Key key) {
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
        active.acquire();
    }

    return active;
}

//从MemoryCache中查找
private EngineResource<?> loadFromCache(Key key) {
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
        cached.acquire();
        activeResources.activate(key, cached);
    }
    return cached;
}

private EngineResource<?> getEngineResourceFromCache(Key key) {
    //首先将找到key对应的cache，并移除，为什么这么做，我们看上一个函数，如果找到的cache不为null，也就是memoryCache缓存命中了，那么需要将这个cache移动到ActiveResource，所以就要将cache从MemoryCache中移除
    Resource<?> cached = cache.remove(key);

    final EngineResource<?> result;
    if (cached == null) {
        result = null;
    } else if (cached instanceof EngineResource) {
        // Save an object allocation if we've cached an EngineResource (the typical case).
        result = (EngineResource<?>) cached;
    } else {
        result =
            new EngineResource<>(
            cached, /*isMemoryCacheable=*/ true, /*isRecyclable=*/ true, key, /*listener=*/ this);
    }
    return result;
}
```

以上就是在有内存缓存或者弱应用缓存情况下，并且能命中的情况分析了。对应我们上面流程图的红色边框里面的部分。

![image-20220322095603727](Glide%E7%9A%84%E7%BC%93%E5%AD%98.assets/image-20220322095603727.png)

##### 磁盘缓存知识了解

到这里如果都还加载不到的话，那么就需要去磁盘获取数据了。磁盘读写也是用的LRU算法。但是这个和内存的LRU算法有一点小区别。为什么呢？因为内存缓存是我们运行的时候，程序加载内存里面的资源，可以直接通过一个`LinkedHashMap`去实现。但是磁盘不同，我总不可能吧所有磁盘的资源读出来然后加载在内存里面吧，这样的话，肯定会引发oom了。那么Glide是怎么做磁盘的LRU的呢？

Glide 是使用一个**日志清单文件**来保存这种顺序，`DiskLruCache` 在 APP 第一次安装时会在缓存文件夹下创建一个 **journal** 日志文件来记录图片的添加、删除、读取等等操作，后面每次打开 APP 都会读取这个文件，把其中记录下来的缓存文件名读取到 `LinkedHashMap` 中，后面每次对图片的操作不仅是操作这个 `LinkedHashMap` 还要记录在 journal 文件中. journal 文件内容如下图：

[![journal 文件内容](Glide%E7%9A%84%E7%BC%93%E5%AD%98.assets/journal.png)](https://github.com/0xZhangKe/Glide-note/blob/master/static/journal.png)

开头的 `libcore.io.DiskLruCache` 是魔数，用来标识文件，后面的三个 1 是版本号 `valueCount` 等等，再往下就是图片的操作日志了。

DIRTY、CLEAN 代表操作类型，除了这两个还有 REMOVE 以及READ，紧接着的一长串字符串是文件的 Key，由 `SafeKeyGenerator` 类生成，是由图片的宽、高、加密解码器等等生成的 SHA-256 散列码后面的数字是图片大小。

根据这个字符串就可以在同目录下找到对应的图片缓存文件，那么打开缓存文件夹即可看到上面日志中记录的文件：

[![缓存文件列表](Glide%E7%9A%84%E7%BC%93%E5%AD%98.assets/disk_cache_file_list.png)](https://github.com/0xZhangKe/Glide-note/blob/master/static/disk_cache_file_list.png)

可以看到日志文件中记录的缓存文件就在这个文件夹下面。

由于涉及到磁盘缓存的外部排序问题，所以相对而言磁盘缓存比较复杂。

磁盘的缓存的前置知识就讲这么多。接下来还是要去看磁盘是怎么进行读写的。在上一篇Glide的流程分析博客中，我们Engine的load方法最终会走到

##### ResourceCacheGenerator#startNext

`ResourceCacheGenerator#startNext`方法里面所以，我们直接看这个方法

```java
public boolean startNext() {
    //首先这里调用了getCacheKeys(),这里返回的不是空，而是一个GlideUrl结果的列表
    List<Key> sourceIds = helper.getCacheKeys();//注意这段代码会将key保存下来，供下面的一些操作使用
    if (sourceIds.isEmpty()) {
        return false;
    }
    // 获得了三个可以到达的registeredResourceClasses
    // GifDrawable、Bitmap、BitmapDrawable
    List<Class<?>> resourceClasses = helper.getRegisteredResourceClasses();
    if (resourceClasses.isEmpty()) {
        if (File.class.equals(helper.getTranscodeClass())) {
            return false;
        }
        throw new IllegalStateException(
            "Failed to find any load path from "
            + helper.getModelClass()
            + " to "
            + helper.getTranscodeClass());
    }
    while (modelLoaders == null || !hasNextModelLoader()) {
        resourceClassIndex++;
        if (resourceClassIndex >= resourceClasses.size()) {
            sourceIdIndex++;
            if (sourceIdIndex >= sourceIds.size()) {
                //如果是第一次请求，那么就会在这里返回，后面的就不会去执行了
                return false;
            }
            resourceClassIndex = 0;
        }

		
        Key sourceId = sourceIds.get(sourceIdIndex);
        Class<?> resourceClass = resourceClasses.get(resourceClassIndex);
        Transformation<?> transformation = helper.getTransformation(resourceClass);
        // PMD.AvoidInstantiatingObjectsInLoops Each iteration is comparatively expensive anyway,
        // we only run until the first one succeeds, the loop runs for only a limited
        // number of iterations on the order of 10-20 in the worst case.
        //假设有缓存，那么就需要构建一个ResourceCacheKey
        currentKey =
            new ResourceCacheKey( // NOPMD AvoidInstantiatingObjectsInLoops
            helper.getArrayPool(),
            sourceId,
            helper.getSignature(),
            helper.getWidth(),
            helper.getHeight(),
            transformation,
            resourceClass,
            helper.getOptions());
        //从磁盘中找到key对应的文件
        cacheFile = helper.getDiskCache().get(currentKey);
        //如果找到对应的文件，那么跳出循环
        if (cacheFile != null) {
            sourceKey = sourceId;
            modelLoaders = helper.getModelLoaders(cacheFile);
            modelLoaderIndex = 0;
        }
    }

    //找到缓存文件后，就会执行到这里
    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
        ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
        loadData =
            modelLoader.buildLoadData(
            cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
        if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
            started = true;
            loadData.fetcher.loadData(helper.getPriority(), this);
        }
    }
    return started;
}
```

我们先看怎么从磁盘中去取缓存的。

首先先构造一个`ResourceCacheKey`，然后通过 `helper.getDiskCache().get()`。这里的`getDiskcache`创建的是`DiskLruCacheWrapper`，所以get方法最终调用的是`DiskLruCacheWrapper`的get方法。这就是获取磁盘缓存的内容了。以上分析完的部分都是读取缓存的部分。那么写缓存的部分在哪里呢？思考一个问题，怎么样才能写缓存？肯定是有数据之后才能写缓存啊。所以我们直接跳到有数据之后，怎么去写缓存。至于怎么获取数据，在上一篇博客中已经讲解的很清楚了，这里就不在重复了。

##### SourceGenerator#onDataReady()

```java
@Override
public void onDataReady(Object data) {
    DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
        dataToCache = data;
        // We might be being called back on someone else's thread. Before doing anything, we should
        // reschedule to get back onto Glide's thread.
        cb.reschedule();
    } else {
        cb.onDataFetcherReady(
            loadData.sourceKey,
            data,
            loadData.fetcher,
            loadData.fetcher.getDataSource(),
            originalKey);
    }
}
```

我们解读一下`onDataReady`里面的代码。首先，获取`DiskCacheStrategy`判断能不能被缓存，这里的判断代码在`SourceGenerator.startNext()`中出现过，显然是可以的。然后将data保存到`dataToCache`，并调用`cb.reschedule()`。
我们在前面分析过，该方法的作用就是将`DecodeJob`提交到Glide的source线程池中。然后执行`DecodeJob.run()`方法，经过`runWrapped()`、 `runGenerators()`方法后，又回到了`SourceGenerator.startNext()`方法。

在方法的开头，会判断`dataToCache`是否为空，此时显然不为空，所以会调用`cacheData(Object)`方法进行data的缓存处理。缓存完毕后，会为该缓存文件生成一个`SourceCacheGenerator`。然后在`startNext()`方法中会直接调用该变量进行加载。

```java
//#SourceGenerator.java
public boolean startNext() {
    //现在是第二次执行到这里
    if (dataToCache != null) {
        Object data = dataToCache;
        dataToCache = null;
        cacheData(data);
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
        return true;
    }
    //省略代码
}

private void cacheData(Object dataToCache) {
    long startTime = LogTime.getLogTime();
    try {
        Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
        DataCacheWriter<Object> writer =
            new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
        originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
        //写缓存，写入的是原始的数据
        helper.getDiskCache().put(originalKey, writer);

    } finally {
        loadData.fetcher.cleanup();
    }

    sourceCacheGenerator =
        new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
}
```

现在我们缓存文件也写完了，那么就会回调DataCacheGenerator的startNext方法

##### DataCacheGenerator#startNext

```java
public boolean startNext() {
    while (modelLoaders == null || !hasNextModelLoader()) {
      sourceIdIndex++;
      if (sourceIdIndex >= cacheKeys.size()) {
        return false;
      }

      Key sourceId = cacheKeys.get(sourceIdIndex);
      // PMD.AvoidInstantiatingObjectsInLoops The loop iterates a limited number of times
      // and the actions it performs are much more expensive than a single allocation.
      @SuppressWarnings("PMD.AvoidInstantiatingObjectsInLoops")
      Key originalKey = new DataCacheKey(sourceId, helper.getSignature());//获取缓存的原始资源文件，这时候获取到的就不会为空了
      cacheFile = helper.getDiskCache().get(originalKey);
      if (cacheFile != null) {
        this.sourceKey = sourceId;
        modelLoaders = helper.getModelLoaders(cacheFile);
        modelLoaderIndex = 0;
      }
    }

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      ModelLoader<File, ?> modelLoader = modelLoaders.get(modelLoaderIndex++);
      loadData =
          modelLoader.buildLoadData(
              cacheFile, helper.getWidth(), helper.getHeight(), helper.getOptions());
      if (loadData != null && helper.hasLoadPath(loadData.fetcher.getDataClass())) {
        started = true;
        loadData.fetcher.loadData(helper.getPriority(), this);//最终会回调到SourceGenerator#onDataFetcherReady
      }
    }
    return started;
  }
```

我们得到数据之后需要进行一定的解码之后才会缓存，这一段代码在哪里呢？在`DecodeJob.onResourceDecoded`，在解码完成之后，我们需要进行磁盘的缓存。注意上一步有一个缓存的操作写的是原始的数据。现在我们需要的是解码之后的数据。

##### DecodeJob#onResourceDecoded

```java
<Z> Resource<Z> onResourceDecoded(DataSource dataSource, @NonNull Resource<Z> decoded) {
    @SuppressWarnings("unchecked")
    Class<Z> resourceSubClass = (Class<Z>) decoded.get().getClass();
    Transformation<Z> appliedTransformation = null;
    Resource<Z> transformed = decoded;
    if (dataSource != DataSource.RESOURCE_DISK_CACHE) {
        appliedTransformation = decodeHelper.getTransformation(resourceSubClass);
        transformed = appliedTransformation.transform(glideContext, decoded, width, height);
    }
    // TODO: Make this the responsibility of the Transformation.
    if (!decoded.equals(transformed)) {
        decoded.recycle();
    }

    final EncodeStrategy encodeStrategy;
    final ResourceEncoder<Z> encoder;
    if (decodeHelper.isResourceEncoderAvailable(transformed)) {
        // encoder就是BitmapEncoder
        encoder = decodeHelper.getResultEncoder(transformed);
        encodeStrategy = encoder.getEncodeStrategy(options);
    } else {
        encoder = null;
        encodeStrategy = EncodeStrategy.NONE;
    }

    Resource<Z> result = transformed;
    boolean isFromAlternateCacheKey = !decodeHelper.isSourceKey(currentSourceKey);
    if (diskCacheStrategy.isResourceCacheable(
        isFromAlternateCacheKey, dataSource, encodeStrategy)) {
        if (encoder == null) {
            throw new Registry.NoResultEncoderAvailableException(transformed.get().getClass());
        }
        final Key key;
        switch (encodeStrategy) {
            case SOURCE:
                key = new DataCacheKey(currentSourceKey, signature);
                break;
            case TRANSFORMED:
                key =
                    new ResourceCacheKey(
                    decodeHelper.getArrayPool(),
                    currentSourceKey,
                    signature,
                    width,
                    height,
                    appliedTransformation,
                    resourceSubClass,
                    options);
                break;
            default:
                throw new IllegalArgumentException("Unknown strategy: " + encodeStrategy);
        }

        LockedResource<Z> lockedResult = LockedResource.obtain(transformed);
        deferredEncodeManager.init(key, encoder, lockedResult);
        result = lockedResult;
    }
    return result;
}
```

从上面的源码中可以看到，当`dataSource != DataSource.RESOURCE_DISK_CACHE`时会进行transform。哦～，这是因为resource cache肯定已经经历过transform了，所以就不用重新来一遍了。

然后是此过程中的磁盘缓存过程，影响的因素有`encodeStrategy`、`DiskCacheStrategy.isResourceCacheable`：

1. encodeStrategy
   根据resource数据的类型来判断，如果是`Bitmap`或`BitmapDrawable`，那么就是`TRANSFORMED`；
   如果是`GifDrawable`，那么就是`SOURCE`。
   详细请看`BitmapEncoder`、`BitmapDrawableEncoder`和`GifDrawableEncoder`类
2. DiskCacheStrategy.isResourceCacheable
   `isFromAlternateCacheKey`搜了一遍源码，只有一个没有使用过的类`BaseGlideUrlLoader`中发现了痕迹，还是一个空集合实现，没有其他任何位置在使用，所以此处可以简单理解的该参数一直为false。
   也就是说，只有dataSource为`DataSource.LOCAL`且encodeStrategy为`EncodeStrategy.TRANSFORMED`时，才允许缓存。换句话说，只有本地的resource数据为`Bitmap`或`BitmapDrawable`的资源才可以缓存。

最后，如果可以缓存，会初始化一个`deferredEncodeManager`，在展示resource资源后会调用此对象进行磁盘缓存的写入。写入的代码如下：

##### DecodeJob#notifyEncodeAndRelease

```java
// DecodeJob.java
//
// deferredEncodeManager.init(key, encoder, LockedResource.obtain(transformed));
private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
  if (resource instanceof Initializable) {
    ((Initializable) resource).initialize();
  }

  Resource<R> result = resource;
  LockedResource<R> lockedResource = null;
  if (deferredEncodeManager.hasResourceToEncode()) {
    lockedResource = LockedResource.obtain(resource);
    result = lockedResource;
  }

  notifyComplete(result, dataSource);

  stage = Stage.ENCODE;
  try {
    if (deferredEncodeManager.hasResourceToEncode()) {
      deferredEncodeManager.encode(diskCacheProvider, options);
    }
  } finally {
    if (lockedResource != null) {
      lockedResource.unlock();
    }
  }
  // Call onEncodeComplete outside the finally block so that it's not called if the encode process
  // throws.
  onEncodeComplete();
}

// DecodeJob#DeferredEncodeManager
private static class DeferredEncodeManager<Z> {
  private Key key;
  private ResourceEncoder<Z> encoder;
  private LockedResource<Z> toEncode;

  @Synthetic
  DeferredEncodeManager() { }

  // We just need the encoder and resource type to match, which this will enforce.
  @SuppressWarnings("unchecked")
  <X> void init(Key key, ResourceEncoder<X> encoder, LockedResource<X> toEncode) {
    this.key = key;
    this.encoder = (ResourceEncoder<Z>) encoder;
    this.toEncode = (LockedResource<Z>) toEncode;
  }

  void encode(DiskCacheProvider diskCacheProvider, Options options) {
    GlideTrace.beginSection("DecodeJob.encode");
    try {
      diskCacheProvider.getDiskCache().put(key,
          new DataCacheWriter<>(encoder, toEncode, options));
    } finally {
      toEncode.unlock();
      GlideTrace.endSection();
    }
  }

  boolean hasResourceToEncode() {
    return toEncode != null;
  }

  void clear() {
    key = null;
    encoder = null;
    toEncode = null;
  }
}
```

至此，磁盘缓存的读写都已经完毕。剩下的就是内存缓存的两个层次了。我们回到`DecodeJob.notifyEncodeAndRelease`方法中，经过`notifyComplete`、`EngineJob.onResourceReady`、`notifyCallbacksOfResult`方法中。
在该方法中一方面会将原始的resource包装成一个`EngineResource`，然后通过回调传给`Engine.onEngineJobComplete`，在这里会将资源保持在active resource中：

##### DecodeJob#onEngineJobComplete

```java
@Override
public synchronized void onEngineJobComplete(
    EngineJob<?> engineJob, Key key, EngineResource<?> resource) {
  // A null resource indicates that the load failed, usually due to an exception.
  if (resource != null) {
    resource.setResourceListener(key, this);

    if (resource.isCacheable()) {
        //将资源缓存到activeResources中，
      activeResources.activate(key, resource);
    }
  }

  jobs.removeIfCurrent(key, engineJob);
}
```

以上就是关于Glide的缓存的全部内容了。