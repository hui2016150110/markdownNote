### Jetpack之Hilt的使用

#### 简单使用

```kotlin
@AndroidEntryPoint
class GalleryFragment : Fragment() {
	
    @Inject
    lateinit var viewModel: GalleryViewModel   //至于需要被依赖注入初始化的变量，不能是私有变量
     override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
       viewModel.xxx()
    }
}
```

比如这里，fragment中有一个viewModel，需要被初始化，给它添加@Inject注解。表示我们通过依赖注入的方式来初始化这个变量。这样就能够直接使用了吗？那肯定是不行的，因为还没有对GalleryViewModel进行声明。那么怎么对GalleryViewModel进行声明呢？

```kotlin
@HiltViewModel
class GalleryViewModel @Inject constructor(
    
) 
```

需要在构造函数前面也加上@Inject注解，这样就可以了。这就是Hilt的最简单的用法，在我们项目中肯定不可能这么简单的使用的。因为在fragment中一般都会有特定的方法去初始化viewmodel，比如通过`by viewModels()`的方式去初始化viewmodel。而且一般来说，viewmodel的构造函数一般都是有参数的构造函数。因为mvvm中有一层Repository层，viewmodel取数据都是去Repository中取得。所以如果有参数的构造函数，那么应该如何去初始化？

#### 有参构造函数的初始化

比如GalleryViewModel中有一个UnsplashRepository类型的参数，如下。那么要如何去初始化呢？

```kotlin
@HiltViewModel
class GalleryViewModel @Inject constructor(
    private val repository: UnsplashRepository
) : ViewModel() {
    
}
```

仔细想想，刚才我们是给ViewModel提供了一个@Inject的注解，然后Hilt就自动去帮助我们去实例化viewModel了。现在的场景是什么？在初始化viewmodel的时候有一个参数，这个参数Hilt不知道哪里去实例化。v**iewmodel是依赖repository的，要对viewmodel通过依赖注入的方式实现初始化，那么Hilt就必须要知道怎么去初始化viewmodel里面的参数**。仔细想想是不是这个道理？打个比方，我要做一台电脑（viewmodel），那么我必须知道主机，显示器等等（viewmodel参数）怎么做，我才能做出一台电脑。

那现在应该怎么做？答案很简单，让Hilt知道怎么去初始化repository。刚刚是如何让hilt知道怎么初始化viewmodel的？是不是在构造函数的时候添加了@Inject注解？所以同理，那么只需要给repository的构造函数添加@Inject就可以了。

如下：

```kotlin
class UnsplashRepository @Inject constructor() {}
```

总结一下上面的有参构造函数的依赖注入：**要对有参构造函数实现依赖注入，那么首先它的每一个参数要支持依赖注入的方式去初始化**。

#### Hilt模块

看了上面的例子，有一些好奇的同学就想了，如果有参构造函数中的参数有接口类型的怎么办？我们不可能通过构造函数注入接口吧。还有，如果这个有参构造函数参数的类型是第三方的库怎么办？我们不可能改变让第三方库也支持依赖注入的构造函数吧。不着急，这些情况Hilt的开发者都有考虑到，并且对上面这些情况都做了支持。那就是利用Hilt模块去做。下面通过例子来讲解Hilt模块怎么使用

还是接着上面的这个例子继续讲解。我们知道Repository是负责提供数据的，这些数据有可能是本地数据，也有可能是网络数据。所以在初始化的Repository的时候肯定是有参数的。如果在UnsplashRepository的构造函数里，有一个参数，他是接口类型如下。

```kotlin
class UnsplashRepository @Inject constructor(private val service: UnsplashService) {
    
}

interface UnsplashService {

    @GET("search/photos")
    suspend fun searchPhotos(
        @Query("query") query: String,
        @Query("page") page: Int,
        @Query("per_page") perPage: Int,
        @Query("client_id") clientId: String = BuildConfig.UNSPLASH_ACCESS_KEY
    ): UnsplashSearchResponse

    companion object {
        private const val BASE_URL = "https://api.unsplash.com/"

        fun create(): UnsplashService {
            val logger = HttpLoggingInterceptor().apply { level = Level.BASIC }

            val client = OkHttpClient.Builder()
                .addInterceptor(logger)
                .build()

            return Retrofit.Builder()
                .baseUrl(BASE_URL)
                .client(client)
                .addConverterFactory(GsonConverterFactory.create())
                .build()
                .create(UnsplashService::class.java)
        }
    }
}
```

在UnsplashRepository中有一个依赖UnsplashService，这个依赖是interface类型的。所以不能通过上面有参构造函数的方式去初始化UnsplashService。这时候需要用Hilt模块去实现了。如下：

```kotlin
@InstallIn(SingletonComponent::class)
@Module
class NetworkModule {

    @Singleton
    @Provides
    fun provideUnsplashService(): UnsplashService {
        return UnsplashService.create()
    }
}
```







#### 为 Android 类生成的组件

来看一下@InstallIn这个注解，就是安装到的意思，那安装到哪里？安装到他后面的这个组件里面(SingletonComponent::class)

Hilt 提供了以下组件：

| Hilt 组件                   | 注入器面向的对象                           |
| :-------------------------- | :----------------------------------------- |
| `SingletonComponent`        | `Application`                              |
| `ActivityRetainedComponent` | `ViewModel`                                |
| `ActivityComponent`         | `Activity`                                 |
| `FragmentComponent`         | `Fragment`                                 |
| `ViewComponent`             | `View`                                     |
| `ViewWithFragmentComponent` | 带有 `@WithFragmentBindings` 注释的 `View` |
| `ServiceComponent`          | `Service`                                  |

#### 组件生命周期

Hilt 会按照相应 Android 类的生命周期自动创建和销毁生成的组件类的实例。

| 生成的组件                  | 创建时机                 | 销毁时机                  |
| :-------------------------- | :----------------------- | :------------------------ |
| `SingletonComponent`        | `Application#onCreate()` | `Application#onDestroy()` |
| `ActivityRetainedComponent` | `Activity#onCreate()`    | `Activity#onDestroy()`    |
| `ActivityComponent`         | `Activity#onCreate()`    | `Activity#onDestroy()`    |
| `FragmentComponent`         | `Fragment#onAttach()`    | `Fragment#onDestroy()`    |
| `ViewComponent`             | `View#super()`           | 视图销毁时                |
| `ViewWithFragmentComponent` | `View#super()`           | 视图销毁时                |
| `ServiceComponent`          | `Service#onCreate()`     | `Service#onDestroy()`     |

#### 组件作用域

**默认情况下，Hilt 中的所有绑定都未限定作用域。这意味着，每当应用请求绑定时，Hilt 都会创建所需类型的一个新实例。**

**不过，Hilt 也允许将绑定的作用域限定为特定组件。Hilt 只为绑定作用域限定到的组件的每个实例创建一次限定作用域的绑定，对该绑定的所有请求共享同一实例。**

下表列出了生成的每个组件的作用域注释：

| Android 类                                 | 生成的组件                  | 作用域                   |
| :----------------------------------------- | :-------------------------- | :----------------------- |
| `Application`                              | `SingletonComponent`        | `@Singleton`             |
| `View Model`                               | `ActivityRetainedComponent` | `@ActivityRetainedScope` |
| `Activity`                                 | `ActivityComponent`         | `@ActivityScoped`        |
| `Fragment`                                 | `FragmentComponent`         | `@FragmentScoped`        |
| `View`                                     | `ViewComponent`             | `@ViewScoped`            |
| 带有 `@WithFragmentBindings` 注释的 `View` | `ViewWithFragmentComponent` | `@ViewScoped`            |
| `Service`                                  | `ServiceComponent`          | `@ServiceScoped`         |

在本例中，如果您使用 `@ActivityScoped` 将 `AnalyticsAdapter` 的作用域限定为 `ActivityComponent`，Hilt 会在相应 Activity 的整个生命周期内提供 `AnalyticsAdapter` 的同一实例：

**注意：将绑定的作用域限定为某个组件的成本可能很高，因为提供的对象在该组件被销毁之前一直保留在内存中。请在应用中尽量少用限定作用域的绑定。如果绑定的内部状态要求在某一作用域内使用同一实例，或者绑定的创建成本很高，那么将绑定的作用域限定为某个组件是一种恰当的做法。**

### 组件层次结构

将模块安装到组件后，其绑定就可以用作该组件中其他绑定的依赖项，也可以用作组件层次结构中该组件下的任何子组件中其他绑定的依赖项：

<img src="Jetpack%E4%B9%8BHilt%E7%9A%84%E4%BD%BF%E7%94%A8.assets/hilt-hierarchy.svg" style="zoom:50%;" />