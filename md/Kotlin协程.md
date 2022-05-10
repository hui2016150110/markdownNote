### Kotlin协程

  Kotlin协程（**本文讲解的协程都是基于Kotlin讲解的，其他语言的协程不在本文章的讨论范围**）目前很流行的一款用于异步任务处理的库，都知道它处理异步任务特别好用，但是很少人去探究它背后的原理。还有一点，由于它是用于处理异步任务的，很多人将协程与线程做对比，也有一些人将协程与Rxjava做对比。这篇文章将从最简单的用法开始，层层递进的讲解以下知识点：

-   如何使用使用协程，以及协程中的一些重要概念
-   协程怎么处理异步任务和并发任务
-   挂起函数是什么
-   协程底层是怎么实现挂起-恢复的
-   协程是怎么做线程切换的

#### 如何使用使用协程，以及协程中的一些重要概念

首先先介绍一下怎么开启一个协程，在Android开发中，如果是在Activity或者Fragment中，那么可以通过以下这种方式开启一个协程。

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch{
            //这里面就是协程
        }
    }
}
```

然而我们肯定不止在`Activity`或者`Fragment`中去使用协程，那么其他地方怎么去开启协程呢？要搞清楚这一点，就要知道`lifecycleScope`是什么，`lifecycleScope`是一个实现了`CoroutineScope`类的一个实例。`CoroutineScope`翻译过来就是**协程作用域**，到这里我们就清楚了，**要开启一个协程，首先就要有一个协程作用域。通过协程作用域的实例去启动一个协程。** 在Android开发中呢，很多时候不需要我们自己创建协程作用域，因为android中有很多拓展属性。比方上面说的`Activity`和`Fragment`中有`lifecycleScope`，`ViewModel`中有`viewModelScope`，可以直接使用这些拓展属性去开启一个协程。那其他地方怎么去创建一个协程作用域呢？首先就是可以通过`MainScope()`去创建一个主线程下协程作用域，还有可以通过`CoroutineScope(Dispatchers.IO)`去创建一个IO线程下的协程作用域。如下

```kotlin
// demo1
val scope = MainScope()
scope.launch{
    Log.d(TAG,Thread.currentThread().name)  // 打印main
}

// demo2
val scope2 = CoroutineScope(Dispatchers.IO)
scope2.launch {
    Log.d(TAG,Thread.currentThread().name) // 打印DefaultDispatcher-worker-1
}
```

上面的这段代码还有一个地方没有讲清楚`CoroutineScope(Dispatchers.IO)`里面的参数是什么？其实`CoroutineScope(context: CoroutineContext)`接收的是一个`CoroutineContext`实例，`CoroutineContext`翻译过来就是**协程的上下文**的意思。

**协程上下文是各种不同元素的集合。其中元素包含了了一个`CoroutineDispatcher`**，即**协程调度器**。**它确定了相关的协程在哪个线程或哪些线程上执行。协程调度器可以将协程限制在一个特定的线程执行，或将它分派到一个线程池，亦或是让它不受限地运行。** 比如上述的例子通过demo1里面的`MainScope()`协程作用域开启的协程，协程是运行在主线程里面的。demo2的协程就是运行在io线程里面的。即使是在通过`MainScope()`开启的协程，依旧可以指定线程。啥意思，看如下这个例子，**launch里面多了Dispatchers.IO这个参数**

```kotlin
val scope = MainScope()
val job = scope.launch(Dispatchers.IO){
    Log.d(TAG,Thread.currentThread().name) //打印 DefaultDispatcher-worker-1
}
```

这里面打印的就不是主线程了，而是IO线程了。看到这里明白了吗？**协程调度器才是真正决定协程在哪个线程运行的关键**，而协程作用域只是给这个协程提供了一个生命周期的管理而已。它并不能真正决定协程运行在哪一个线程。那么demo1打印main的现象怎么解释？因为launch函数如果不传**协程的上下文**，它就默认是协程作用域里面的上下文，而`MainScope()`默认的上下文里面的调度器就是`Dispatchers.Main`.

总结对比一下上面讲述的几个概念： **协程作用域：主要负责管理协程的生命周期。** **协程上下文：由各种元素组成，其中一个元素是协程调度器。** **协程调度器：定了相关的协程在哪个线程或哪些线程上执行。**

#### 协程如何处理异步任务和并发任务

上面说了这么多概念，好像很厉害的样子，但是听完之后也就听完了，啥也没学会。比如他如何处理异步任务？就一个普通的场景，去网络上请求数据，然后在前台显示。协程里面该怎么做，如下

```kotlin
class MainActivity : AppCompatActivity() {

    val TAG = MainActivity::class.java.name
    var text = "hello"
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 步骤一，通过lifecycleScope开启一个协程
        lifecycleScope.launch{
            //步骤二 调用耗时任务
            changeText()
            //步骤三 打印结果
            Log.d(TAG,text) // 最终打印结果 hello Coroutine，这段代码在Main线程
        }

    }

    // changeText 在IO线程模拟一个耗时任务，注意这里的suspend关键字，标识这个函数是挂起函数，什么是挂起函数后面会讲
    private suspend fun changeText(){
        // 通过withContext将协程运行的线程切换到IO线程，然后在IO线程里面做耗时处理，并改变text
        withContext(Dispatchers.IO){
            delay(1000)
            text = "$text Coroutine"
        }
    }
}
```

以上就是简单处理一个耗时任务的例子。看上去是不是很神奇，明明切换了线程，`Log.d(TAG,text)`这段代码不会先执行吗？可以肯定的告诉你，不会。这就是协程相比于线程的优势，**用同步的代码方式去完成异步任务**。而能完成这一切都与挂起函数有关。这是简单的任务，如果是多任务呢？比如说，任务一在IO线程任务二在UI线程，任务三又在IO线程任务四又回来了UI线程：

```kotlin
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch{
            task1()  // io线程执行耗时任务
            task2()  // ui线程执行界面更新
            task3()  // io线程执行耗时任务
            task4()  // ui线程执行界面更新
        }

    }
    
    private suspend fun task1(){
        withContext(Dispatchers.IO){
            delay(1000)
        }
    }
    
    private suspend fun task2(){
        withContext(Dispatchers.Main){
        }
    }
    
    private suspend fun task3(){
        withContext(Dispatchers.IO){
            delay(1000)
        }
    }
    
    private suspend fun task4(){
        withContext(Dispatchers.Main){
        }
    }
```

task1->task2->task3->task4会依次按照顺序执行。没有回调函数，直接明了。<br/>
还有一种比较复杂的情况就是，如果a，b，c三个任务，a，b任务的结果用于c任务的参数，那应该怎么做？我们先想一下如果是不用协程我们改怎么做，很多人说用rxjava。确实rxjava可以比较简单的实现我们上面的功能。如果只用线程来做的话是不是很麻烦，因为我们不知道任务a，b哪一个更快或者哪一个更忙，这样的任务用线程来管控的话，会非常麻烦，所以我们在代码里面可以会先a执行完，在执行b，然后再执行c，这样做的话，效率就会很低了。而且任务如果有10个呢（这当然是比较极端的情况了），那么当当是用线程写起来可读性就很差了。那么在协程中如何去做呢？如下：

```kotlin
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch{
            val time = measureTimeMillis{
                // ① 通过async启动一个协程，async 返回一个 Deferred —— 一个轻量级的非阻塞 future， 这代表了一个将会在稍后提供结果的 promise。你可以使用 .await() 在一个延期的值上得到它的最终结果
                val firstName = async(Dispatchers.IO) { getFirstName() }
                
                val second = async(Dispatchers.IO) { getSecondName() }
                // 通过await()得到结果，getFirstName() 与getSecondName()是并发的
                val name = firstName.await() + " - " +second.await()
                val friend = withContext(Dispatchers.IO){
                    getFriend(name)
                }
                Log.d(TAG,friend)  // John - Tom Amy
            }
            Log.d(TAG,"$time")  // 2000ms右一般2050左右，不到3000ms
        }

    }

    private suspend fun getFirstName():String{
            delay(1000)
            return "John"
    }

    private suspend fun getSecondName():String{
        delay(1000)
        return "Tom"
    }

    private suspend fun getFriend(name:String):String{
        delay(1000)
        return "$name Amy"
    }
```
以上就是协程中如何处理异步和做并发了。

总结： <br/>
1、在Android中，可以开启一个主线程作用域的协程，然后需要切线程的时候通过withContext去切换线程，耗时任务放到IO线程中去执行，并且将耗时任务通过suspend声明为**挂起函数**。完成以上步骤就可以在协程中，用同步方式的代码去实现简单的异步任务了。<br/>
2、如果要并发任务，可以通过async关键字开启一个新的协程，然后之后通过.await()拿到结果。
#### 挂起函数是什么

通过以上的讲解，我们仅仅是知道怎么使用协程，怎么去使用协程完成并发任务，我们还不知道协程为什么能够用同步的代码方式去完成异步任务。要想知道这个，就从一个关键字说起，那就是`suspend`关键字，被`suspend`关键字标记的函数叫`挂起函数`。

首先先来看一下什么是CPS。Suspending functions are implemented via Continuation-Passing-Style (CPS). Every suspending function and suspending lambda has an additional `Continuation` parameter that is implicitly passed to it when it is invoked.

这段话什么意思呢？它的意思是说挂起函数都会经过CPS转换的，CPS转换之后呢会有一个额外的参数Continuation，当调用这个挂起函数的时候，会传递给这个挂起函数。

什么意思呢？来看代码：

```
// 注释1 
private suspend fun getFirstName():String  //kotlin代码
 
 //注释2
 private fun <T> getFirstName(continuation: Continuation<T>):Any? //经过CPS转化后的代码，多了一个Continuation类型的参数，而这个参数就类似一个callback接口的作用
 
 /** 这是Continuation的定义
 *Here is the definition of the standard library interface Continuation
 *(defined in kotlinx.coroutines package), which represents a generic callback:
 */
 interface Continuation<in T> {
   val context: CoroutineContext
   fun resumeWith(result: Result<T>)
}
```

从注释1到注释2的过程，就是CPS的过程。函数类型由原来的 `suspend()->String` 变成了`Continuation->Any?`

那么Continuation是什么，它是一个类似于callback的东西，里面的resumeWith函数，就类似于callBack里面的回调函数，那么这个

Continuation指的是哪一部分呢？

![image-20211230183714920.png](Kotlin%E5%8D%8F%E7%A8%8B.assets/60670a781ad34efc8c0c82c6411d147dtplv-k3u1fbpfcp-watermark.image)</br>
它大概会转换成下面这个样子

```kotlin
 lifecycleScope.launch{
            task1(object: Continuation<Unit>{
                override fun resumeWith(result: Result<Unit>) {
                    // 也就是说，等到task1，执行完成之后才会执行到task2，task3与task4
                    task2()  // ui线程执行界面更新
                    task3()  // io线程执行耗时任务
                    task4()  // ui线程执行界面更新
                }

                override val context: CoroutineContext
                    get() = TODO("Not yet implemented")

            })  // io线程执行耗时任务

        }
```

以上大概就是有关挂起函数的讲解了。

简单总结一下：在kotlin中，如果用suspend声明的函数，称为挂起函数。挂起函数的原理其实就是CPS转换。挂起函数并没有切换线程的功能，将函数声明为挂起函数，只是做一个标记，让编译器去做CPS转换，**这个CPS转换对开发者来说是无感知的，所以我们能以同步的方式去实现异步的任务**。



#### 协程如何去实现挂起-恢复的

通过以上的讲解，我们知道了协程是如何工作了，但是我们还是不知道协程如果去实现这些功能的，首先看一段代码，跟着这一段代码，我们一步一步去讲解协程是如何实现挂起-恢复的。

```kotlin

    fun testCoroutine() {
        lifecycleScope.launch {
            val firstName = getFirstName()
            Log.d(TAG, firstName)
        }
        Log.d(TAG, "主线程继续执行")
    }

    private suspend fun getFirstName(): String {
        var name = ""
        withContext(Dispatchers.IO) {
            delay(1000)
            name = "hello"
        }
        return name
    }
```

以上代码的执行步骤如下：
1、在主线程中开启一个协程

2、通过withContext切换了线程去做耗时任务，同时主线程打印“主线程继续执行”

3、耗时任务执行完成，并且在主线程将值赋给firstName，主线程打印firstname

如下：
![image-20211231143751189](Kotlin%E5%8D%8F%E7%A8%8B.assets/image-20211231143751189.png)

**在执行代码1的时候，在IO线程做耗时任务，这时候主线程的代码块2是不执行的，代码块2被挂起了，但是主线程的代码块3这时候是执行的，代码块1里面的耗时任务执行完成之后，主线程2里面的代码会被恢复，然后继续执行完成**。

现在的难点在于：

协程如何做挂起和恢复。

首先我们将delay()（这个delay()只是代表了耗时任务，但是他会增加我们阅读反编译代码的难度）这段代码删除，然后反编译一下。这里我就不直接贴反编译的代码了，因为直接贴反编译的代码，它的可读性太差了。它反编译之后大概如下：

```java
   public final void testCoroutine() {
      BuildersKt.launch$default((CoroutineScope)LifecycleOwnerKt.getLifecycleScope(this), (CoroutineContext)null, (CoroutineStart)null, (Function2)(new Function2((Continuation)null) {
         int label;

         @Nullable
         public final Object invokeSuspend(@NotNull Object result) {  // 状态机状态的切换
             
            suspendLable = IntrinsicsKt.getCOROUTINE_SUSPENDED(); // 是否要挂起
            switch(this.label) {
            case 0:
               this.label = 1;
               funtionSuspend = var4.getFirstName(this); //是否为挂起函数，注意这里的this
               if (suspendLable == funtionSuspend) {
                  return suspendLable; //如果是挂起函数，那么直接return，return之后就可以执行 这段代码 Log.d(this.TAG, "主线程继续执行");
               }
               break;
            case 1:
               finalResult = result;
               break;
            default:
               throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
            }

            String firstName = (String)finalResult;

            return Unit.INSTANCE;
         }
      }), 3, (Object)null);
      Log.d(this.TAG, "主线程继续执行");
   }

   private final Object getFirstName(Continuation var1) {
      Object $result = ((<undefinedtype>)$continuation).result;
      Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      final ObjectRef name;
      switch(label) {
      case 0:
         name = "";
         CoroutineContext var10000 = (CoroutineContext)Dispatchers.getIO(); //切换线程
         Function2 var10001 = (Function2)(new Function2((Continuation)null) { 
            int label;
            @Nullable
            public final Object invokeSuspend(@NotNull Object var1) {
               switch(this.label) {
               case 0:
                  name= "hello";
                  return Unit.INSTANCE;
               }
            }
         });
         break;
      case 1:
         name = (ObjectRef)((<undefinedtype>)$continuation).L$0;
         ResultKt.throwOnFailure($result);
         break;
      }
      return name;
   }
```

编译后的代码大概就是这样，这种利用label进行状态判断的代码，也叫状态机机制，其实协程就是通过状态机去实现挂起与恢复的一个过程。在testCoroutine，我们能够清楚的看到，如果在线程中执行了一个挂起函数，那么他就会直接return掉。这里也解释了为什么在执行挂起函数的时候，协程外的主线程会执行了。

![image-20211231162931961](Kotlin%E5%8D%8F%E7%A8%8B.assets/image-20211231162931961.png)

那怎么恢复呢？恢复的代码在反编译的代码中是没有呈现出来的，他其实是通过执行了invokeSuspend函数来进行恢复的。再一次执行invokeSuspend的时候，这时候它的label就不是0了，而是1了，所以他会执行-  `finalResult = result;`然后跳出switch语句，并且执行`String firstName = (String)finalResult;`语句。这样一整个协程的挂起与恢复的流程也就结束了。

总结：协程的挂起与恢复是通过状态机去实现的。**每一个挂起点（挂起函数）都是一种状态，协程恢复只是跳转到下一个状态，挂起点将执行过程分割成多个片段，利用状态机的机制保证各个片段按顺序执行。**

#### 协程是如何做线程切换的：

那么现在还剩下最后一个问题，协程的底层是怎么做线程切换的呢？其实在刚刚的反编译代码中就可以看出，协程它的底层是通过Dispatchers去切换线程的，那么它是怎么切换的呢？要研究这个问题就要从最开始的launch{}的源码开始了，今天当然不会去纠结源码的实现细节，我们知道底层切换线程是怎么做的就好了。

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

第一个参数 **context** 是协程上下文，在讲述协程概念的时候有提到过
第二个参数 **start** 此处我们没有传值则使用默认值，代表启动方式默认值为立即执行。
第三个参数 **block** 是协程真正执行的代码块，即`launch{}`花括号中的代码块。

`launch{}`里面做了什么？

1、创建一个新的协程上下文。

2、再创建一个`Continuation`，默认情况下是`StandaloneCoroutine`

3、启动`Continuation`

首先来看：`newCoroutineContext`

```kotlin
public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
    val combined = coroutineContext + context   // 将launch方法传入的context与CoroutineScope中的context组合起来
    val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined
    return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
        debug + Dispatchers.Default else debug // 如果combined中没有拦截器，会传入一个默认的拦截器，即Dispatchers.Default，是CoroutineDispatcher类型的，
}
```

再来看启动`Continuation`

```kotlin
coroutine.start(start, coroutine, block)

->AbstractCoroutine.start()

->CoroutineStart.invoke()

->block.startCoroutineCancellable()

->createCoroutineUnintercepted(receiver, completion).intercepted().resumeCancellableWith(Result.success(Unit), onCancellation)

```

最终会执行到我们的最后一行。最后一行也是分析的重点。

首先创建一个协程，并链式调用`intercepted()`和`resumeCancellableWith()`方法。createCoroutineUnintercepted()这个方法目前看不到源码的实现，不过不影响我们后面的分析，先看`intercepted()`

```kotlin
public actual fun <T> Continuation<T>.intercepted(): Continuation<T> =
    (this as? ContinuationImpl)?.intercepted() ?: this
    
public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
//上面讲解newCoroutineContext的时候，讲解到有一个默认的Dispatchers.Default是CoroutineDispatcher，所以这里最终会调用到CoroutineDispatcher的interceptContinuation()

public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
     DispatchedContinuation(this, continuation)
```

所以，我们可以发现这里其实就是创建的一个`DispatchedContinuation`，并且将原来的协程放入的`DispatchedContinuation`中。

最后看一下`resumeCancellableWith`这里很明显了，调用的是`DispatchedContinuation`的`resumeCancellableWith`。

```kotlin
    @Suppress("NOTHING_TO_INLINE")
    inline fun resumeCancellableWith(
        result: Result<T>,
        noinline onCancellation: ((cause: Throwable) -> Unit)?
    ) {
        val state = result.toState(onCancellation)
        if (dispatcher.isDispatchNeeded(context)) {
            _state = state
            resumeMode = MODE_CANCELLABLE
            dispatcher.dispatch(context, this)
        } else {
            executeUnconfined(state, MODE_CANCELLABLE) {
                if (!resumeCancelled(state)) {
                    resumeUndispatchedWith(result)
                }
            }
        }
    }
```

来看一下这段代码，如果需要切换线程，那么调用dispatcher.dispatch方法，如果不需要，直接运行在原来的线程上。

那么接下来就是要看一下dispatcher是怎么切换线程的了，`DispatchedContinuation`提供了四种实现，我们接下来只看`Dispatchers.IO`

最终会调用的`ExperimentalCoroutineDispatcher`的dispatch方法

```kotlin
override fun dispatch(context: CoroutineContext, block: Runnable): Unit =
        try {
            coroutineScheduler.dispatch(block)
        } catch (e: RejectedExecutionException) {
            // CoroutineScheduler only rejects execution when it is being closed and this behavior is reserved
            // for testing purposes, so we don't have to worry about cancelling the affected Job here.
            DefaultExecutor.dispatch(context, block)
        }
```

而这里的`coroutineScheduler`是一个`Executor`。看到这里，我们就大概知道协程是怎么做一个线程切换了，它的底层是通过线程池去做线程的切换的。这是`Dispatchers.IO`的实现，如果是`Dispatchers.Main`的，它的底层是通过handler去做的一个线程切换。

到这里协程的知识点讲的也就差不多了。下面总结一下。

#### 总结

1、挂起函数的作用不是做线程切换，它的作用是在编译的时候，让编译器做CPS转换，真正做线程切换的是线程池还有Handler。

2、所谓的挂起-恢复是协程帮助我们去实现的一套机制，是利用CPS转换+状态机的机制去实现的，在其他地方并不能实现，这也就是为什么挂起函数只能在挂起函数里面或者协程中被调用的原因。

3、协程和线程的关系是什么？在kotlin中，协程是运行于线程之上的。因为JVM中，所有的代码都是运行在线程上面的，这一点协程也不例外。**但是协程并<font color = red>不是</font>所谓的轻量级线程**，协程的作用是他能够更加方便的去做一个异步的任务。

4、协程与Rxjava的区别是什么？首先它们是都是异步框架，都能很方便的处理异步任务，区别就是Rxjava会更加强大一点，并且kotlin，java中都能使用，协程只能在kotlin中使用。在协程推出flow之后，协程也变得越来越强大了。