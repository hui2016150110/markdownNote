### MediaPlayer状态图及生命周期

MediaPlayer是Android中的uoge多媒体播放类，我们能通过它控制音视频流或本地音视频资源的播放过程。

这一片博客主要介绍MediaPlayer状态图及生命周期。先看一张官网很经典的MediaPlayer状态机的图片。

![MediaPlayer State diagram](MediaPlayer%E7%8A%B6%E6%80%81%E5%9B%BE%E5%8F%8A%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.assets/mediaplayer_state_diagram.gif)

**其中椭圆代表MediaPlayer驻留状态，弧代表播放器控制且驱动MediaPlayer状态进行过度。有两种类型的弧，单箭头弧表示的是同步函数的调用，双箭头弧表示的是异步函数的调用。**



从上图中我们能够得知MediaPlayer有一下状态

+ #### Idle状态及End状态

  在MediaPlayer创建实例或者调用Reset()函数后，播放器就被创建了，这时播放器就处于Idle（就绪）状态。调用release()函数后，播放器就会变成End(结束)状态，在这两种状态之间的就是MediaPlayer的生命周期。

+ #### Error状态

  在构造的一个新的MediaPlayer获取调用`reset()`函数之后 。上层应用调用等`getCurrentPosition()`， `getDuration()`，`getVideoHeight()`， `getVideoWidth()`，`setAudioAttributes(android.media.AudioAttributes)`， `setLooping(boolean)`， `setVolume(float, float)`，`pause()`，`start()`， `stop()`，`seekTo(long, int)`，`prepare()`或者`prepareAsync()`这些函数都会出错。但是如果这些方法在之后被调用`reset()`，将会回调到用户提供的回调方法 OnErrorListener.onError() ，这时候MediaPlayer就处于Error(错误)状态。所以一旦不再使用MediaPlayer，就需要调用release()函数，以便MediaPlayer资源得到合理的释放

  当MediaPlayer处于End(结束)状态时，它将不能再被使用，这个时候不能再回到MediaPlayer的其他状态，因为本次生命周期已经终止。

  **在一般情况下，由于种种原因一些播放控制操作可能会失败，如不支持的音频/视频格式，缺少隔行扫描的音频/视频，分辨率太高，流超时等原因，等等。因此，错误报告和恢复在这种情况下是非常重要的。有时，由于编程错误，在处于无效状态的情况下调用了一个播放控制操作可能发生。在所有这些错误条件下，内部的播放引擎会调用一个由客户端程序员提供的OnErrorListener.onError()方法。客户端程序员可以通过调用MediaPlayer.setOnErrorListener（android.media.MediaPlayer.OnErrorListener）方法来注册OnErrorListener.**

   一旦发生错误，**MediaPlayer**对象会进入到Error状态。为了重用一个处于Error状态的**MediaPlayer**对象，可以调用**reset()**方法来把这个对象恢复成Idle状态。注册一个**OnErrorListener**来获知内部播放引擎发生的错误是好的编程习惯。在不合法的状态下调用一些方法，如**prepare()，prepareAsync()**和**setDataSource()**方法会抛出**IllegalStateException**异常。

+ #### Initialized状态

  **调用setDataSource(FileDescriptor)方法，或setDataSource(String)方法，或setDataSource(Context，Uri)方法，或setDataSource(FileDescriptor，long，long)方法会使处于Idle状态的对象迁移到Initialized状态。**若当此**MediaPlayer**处于其它的状态下，调用**setDataSource()**方法，会抛出**IllegalStateException**异常。好的编程习惯是不要疏忽了调用**setDataSource()**方法的时候可能会抛出的**IllegalArgumentException**异常和**IOException**异常。

+ #### Prepared状态

  **在开始播放之前，MediaPlayer对象必须要进入Prepared状态。**

  有两种方法（同步和异步）可以使MediaPlayer对象进入Prepared状态：要么调用**prepare()**(**同步**)，此方法返回就表示该**MediaPlayer**对象已经进入了Prepared状态；要么调用**prepareAsync()**(**异步**)，此方法会使此**MediaPlayer**对象进入**Preparing**状态并返回，而内部的播放引擎会继续未完成的准备工作。当同步版本返回时或异步版本的准备工作完全完成时就会调用客户端程序员提供的**OnPreparedListener.onPrepared()**监听方法。可以调用**MediaPlayer.setOnPreparedListener(android.media.MediaPlayer.OnPreparedListener)**方法来注册**OnPreparedListener**.
  注意：Preparing是一个中间状态，在此状态下调用任何具备边影响的方法的结果都是未知的！
  在不合适的状态下调用**prepare()**和**prepareAsync()**方法会抛出**IllegalStateException**异常。当**MediaPlayer**对象处于**Prepared**状态的时候，可以调整音频/视频的属性，如音量，播放时是否一直亮屏，循环播放等。 

+ #### Started状态

  在播放控制开始之前，必须调用start函数并成功返回，MediaPlayer的状态开始由Prepared状态变成Started状态。当处于Started状态时，如果用户实现注册过setOnBufferingUpdateListener，播放器内部会开始回调setOnBufferingUpdateListener.onBufferingUpdate，这个回调函数主要使应用程序保持跟踪音视频流的buffering（缓冲） status，**如果MediaPlayer已经处于Started状态，再调用start()函数是没有任何作用的**。

+ #### Paused状态

  MediaPlayer在播放控制时可以是Paused(暂停)和Stopped(停止)状态的，**且当前的播放时进度可以被调整**，当调用MediaPlayer.pause函数时，MediaPlayer开始由Started状态变成Paused状态，这个从Started状态到Paused状态的过程是瞬间的，**反之在播放器内部是异步过程的**。在状态更新并调用isPlaving函数前，将有一些耗时。已经缓冲过的数据流，也要耗费数秒。
  **当start函数从Paused状态恢复回来时，playback恢复之前暂停时的位置，接着开始播放**，这时MediaPlayer的Paused状态又变成Started状态。如果MediaPlayer已经处于Paused状态，**这时再调用pause函数是没有任何作用的，将保持Paused状态**。

+ #### Stopped状态

  当调用stop函数时，MediaPlayer无论正处于Started、Paused、 Prepared 或 PlaybackCompleted中的哪种状态，都将进入Stopped状态。一旦处于Stopped状态，playback将不能开始，直到重新调用prepare或prepareAsync函数，且处于Prepared状态时才可以开始。

  如果MediaPlayer已经处于Stopped状态了，这时再调用stop函数是没有任何作用的，将保持 Stopped状态。
  在Seek操作完成后，如果事先在MediaPlayer注册了setOnSeekCompleteListener，播放器内部将回调OnSeekCompleteonSeekComplete雨数。当然seekTo函数也可以在其他状态下被调用，如Prepared、Paused及PlaybackCompleted状态。

+ #### PlaybackCompleted状态

  当前播放的位置可以通过getCurrentPosition函数获取，通过getCurrentPosition函数，可以跟踪播放器的播放进度。
  当MediaPlaver播放到数据流的末尾时，一次播放过程完成。在MediaPlaver中事先调用setLooping(boolean)并设置为true，表示循环播放，MediaPlayer依然处于Started状态。如果调用setLooping(boolean)并设置为false(表示不循环播放)，并且事先在MediaPlayer 上注册过 setOnCompletionListener，播放器内部将回调OnCompletiononCompletion 函数，这就表明 MediaPlayer 开始进入PlaybackCompleted(播放完成)状态。当处于PlaybackCompleted 状态时，调用start函数，将重启播放器从头开始播放数据。

以上就是本博客的内容了，了解了MediaPlayer的各种状态，有利于之后的使用。