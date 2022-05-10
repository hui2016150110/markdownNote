MediaCodec讲解

[MediaCodec](https://developer.android.google.cn/reference/android/media/MediaCodec)是Android提供的用于对音视频进行编解码的类，它通过访问底层的codec来实现编解码的功能。是Android media基础框架的一部分，通常和 MediaExtractor, MediaSync, MediaMuxer, MediaCrypto, MediaDrm, Image, Surface和AudioTrack一起使用。

#### MediaCodec支持的数据类型

编解码器支持的数据类型：**压缩的音视频数据，原始音频数据和原始视频数据**。

+ 数据通过ByteBuffers类来表示。

+ 可以设置Surface来获取/呈现原始的视频数据，Surface使用本地的视频buffer，不需要进行ByteBuffers拷贝。可以让编解码器的效率更高。

+ 通常在使用Surface的时候，无法访问原始的视频数据，但是可以使用ImageReader访问解码后的原始视频帧。在使用ByteBuffer的模式下，可以使用Image类和getInput/OutputImage(int)获取原始视频帧。

##### **压缩的音视频数据**

+ 对于视频类型，这通常是一个压缩视频帧。
+ 对于音频数据，这通常是单个访问单元（通常包含由格式类型的指定的几毫秒的音频段（通常包含几毫秒的音频），但是该要求略微放松，因为一个buffer可以包含多个编码的音频访问单元。
+ 在以上两种情况下，buffer都不在任意字节边界上启动或结束，而是在帧/访问单元边界上启动或结束，除非它们被BUFFER_FLAG_PARTIAL_FRAME标记。

##### **原始音频数据**

原始音频buffer包含PCM音频数据的整个帧，这是每个通道按通道顺序的一个样本。每个样本都是一个 AudioFormat#ENCODING_PCM_16BIT。

##### **原始视频数据**

在ByteBuffer模式下，视频buffer根据它们的MediaFormat#KEY_COLOR_FORMAT进行布局。可以从getCodecInfo(). MediaCodecInfo.getCapabilitiesForType.CodecCapability.colorFormats获取支持的颜色格式。视频编解码器可以支持三种颜色格式:

- **native raw video format**: CodecCapabilities.COLOR_FormatSurface，可以与输入/输出的Surface一起使用。
- **flexible YUV buffers** 例如CodecCapabilities.COLOR_FormatYUV420Flexible， 可以使用getInput/OutputImage(int)与输入/输出Surface一起使用，也可以在ByteBuffer模式下使用。
- **other, specific formats:** 通常只支持ByteBuffer模式。有些颜色格式是厂商特有的，其他定义在CodecCapabilities。对于等价于flexible格式的颜色格式，可以使用getInput/OutputImage(int)。

从Build.VERSION_CODES.LOLLIPOP_MR1.开始，所有视频编解码器都支持flexible的YUV 4:2:0 buffer。

#### MediaCodec状态与生命周期

![image-20220411164809020](MediaCodec%E8%AE%B2%E8%A7%A3.assets/image-20220411164809020.png)

MediaCodec生命周期状态分为三种 Stopped、Executing和Released
其中Stopped包含三种子状态 Uninitialized（为初始化状态）、Configured（已配置状态）、Error（异常状态）
Executing也包含三个子状态 Flushed（刷新状态）、Running（运行状态）和EOS（流结束状态）

##### Stopped状态：

+ Uninitialized：当使用工厂方法创建了一个MediaCodec对象，此时处于Uninitialized状态。可以在任何状态调用reset()方法使MediaCodec返回到Uninitialized状态
+ Configured：使用configure(…)方法对MediaCodec进行配置转为Configured状态
+ Error：MediaCodec遇到错误时进入Error状态。错误可能是在队列操作时返回的错误或者异常导致的。

##### Executing状态：

当调用了mediaCodec.start()方法后，就由stopped到Executing状态了，在此状态下，可以通过上面描述的缓冲队列操作来处理数据

+ Flushed：在调用start()方法后MediaCodec立即进入Flushed子状态，此时MediaCodec会拥有所有的缓存。可以在Executing状态的任何时候通过调用flush()方法返回到Flushed子状态。
+ Running：一旦第一个输入缓存（input buffer）被移出队列，MediaCodec就转入Running子状态，这种状态占据了MediaCodec的大部分生命周期。通过调用stop()方法转移到Uninitialized状态。
+ EOS：将一个带有end-of-stream标记的输入buffer入队列时，MediaCodec将转入End-of-Stream子状态。在这种状态下，MediaCodec不再接收之后的输入buffer，但它仍然产生输出buffer直到end-of-stream标记输出。

##### **Released**状态

当使用完MediaCodec后，必须调用release()方法释放其资源。调用 release()方法进入最终的Released状态。

#### 工作原理和基本流程

![image-20220411171046096](MediaCodec%E8%AE%B2%E8%A7%A3.assets/image-20220411171046096.png)

来看这个图片，这张图片就是MediaCodec的工作原理。简单讲一下就是。

+ 数据的生产方（左侧的Client）从input缓冲队列申请empty buffer，然后把要处理的数据，填充到这些empty buffer里面。就是上面的空方块（empty buffer），经过Client（数据生成方）之后，变成了红色实心的方块（有数据的buffer）

+ 这些数据经过Codec处理之后，然后处理后的数据（黄色的方块）放入到右侧output缓冲区队列

+ 消费方Client（右侧Client）从output缓冲区队列申请处理后的buffer，然后进一步处理，最后再将改buffer放回缓冲队列。

接下来，就以解析mp4的视频为例。讲解一下mediaCodec的解码视频轨道的过程。各个功能看文中的代码注释，一些异常处理，这里就暂时不考虑了。

这里大概讲一下mp4解码的过程(代码参考[grafika](https://github.com/google/grafika))

1. 创建MediaExtractor(),并设置数据源，MediaExtractor类，可以用来分离容器中的视频track和音频track。然后选中音频轨道
2. 通过MediaCodec.createDecoderByType解码器
3. 调用decoder的configure与start函数
4. 调用decoder.dequeueInputBuffera得到inputBufIndex
5. decoder.getInputBuffer(inputBufIndex)得到inputBuf
6. 调用 extractor.readSampleData填充inputBuf
7. 调用extractor.advance()移动到下一个样本
8. 调用decoder.dequeueOutputBuffer(mBufferInfo, TIMEOUT_USEC)得到decoderStatus，这里是消费者消费数据
9. 调用decoder.releaseOutputBuffer(decoderStatus,doRender)，释放输出的Buffer空间

```kotlin
package com.example.videodemo

import android.media.MediaCodec
import android.media.MediaExtractor
import android.media.MediaFormat
import android.os.Handler
import android.os.Message
import android.util.Log
import android.view.Surface
import java.io.File
import java.io.FileNotFoundException
import java.io.IOException

class MediaCodecDemo constructor(val path: File, val outputSurface: Surface,val callback: SpeedControlCallback) {
    private val TAG: String = "MediaCodecDemo"

    var mVideoHeight = -1
    var mVideoWidth = -1
    private val mBufferInfo = MediaCodec.BufferInfo() //输出buffer的metadata
    var mLoop = false

    init {

        var extractor: MediaExtractor? = null
        try {
            extractor = MediaExtractor()
            extractor.setDataSource(path.toString())
            val trackIndex = selectVideoTrack(extractor)
            if (trackIndex < 0) {
                throw RuntimeException("找不到视频轨道")
            }
            extractor.selectTrack(trackIndex)
            val format = extractor.getTrackFormat(trackIndex)
            mVideoHeight = format.getInteger(MediaFormat.KEY_HEIGHT)
            mVideoWidth = format.getInteger(MediaFormat.KEY_WIDTH)
        } finally {
            extractor?.release()
        }

    }

    //寻找视频轨道，并返回对应的index
    private fun selectVideoTrack(extractor: MediaExtractor): Int {
        val numTracks = extractor.trackCount
        Log.d("hch","index $numTracks")
        for (index in 0 until numTracks) {
            val format = extractor.getTrackFormat(index)
            val mime = format.getString(MediaFormat.KEY_MIME)
            mime?.let {
                if (it.startsWith("video/")) {
                    Log.d("hch","index $index")
                    return index
                }
            }
        }
        return -1
    }

    private fun play() {
        var extractor: MediaExtractor? = null
        var decoder: MediaCodec? = null
        try {
            extractor = MediaExtractor()
            extractor.setDataSource(path.toString())
            //文件不存在，直接抛出异常
            if (!path.exists()) {
                throw FileNotFoundException("file donot exists")
            }
            val trackIndex = selectVideoTrack(extractor)
            if (trackIndex < 0) {
                throw RuntimeException("video track dont find")
            }
            extractor.selectTrack(trackIndex)
            val format = extractor.getTrackFormat(trackIndex)
            val mine = format.getString(MediaFormat.KEY_MIME)
            decoder = mine?.let { MediaCodec.createDecoderByType(it) }
            decoder?.let {
                decoder?.configure(format, outputSurface, null, 0)
                decoder?.start()
                doFrame(extractor!!, trackIndex, it)
            }

        } catch (e:IOException){

        }
        finally {
            decoder?.stop()
            decoder?.release()
            decoder = null
            extractor?.release()
            extractor = null
        }
    }

    private fun doFrame(extractor: MediaExtractor, trackIndex: Int, decoder: MediaCodec) {
        val TIMEOUT_USEC = 10000L
        var inputChunk = 0
        var inputDone = false
        var outputDone = false
        while (!outputDone) {
            //获取输入buff的空间
            if (!inputDone) {
                //得到inputBufIndex
                val inputBufIndex = decoder.dequeueInputBuffer(TIMEOUT_USEC)
                if (inputBufIndex >= 0) {
                    //得到对应的inputBuffer
                    val inputBuf = decoder.getInputBuffer(inputBufIndex)
                    inputBuf?.let {
                        //数据生成方填充对应的inputBuf
                        val chunkSize = extractor.readSampleData(it, 0)
                        if (chunkSize < 0) {
                            //读到文件末
                            decoder.queueInputBuffer(
                                inputBufIndex,
                                0,
                                0,
                                0L,
                                MediaCodec.BUFFER_FLAG_END_OF_STREAM
                            )
                            //推出循环
                            inputDone = true
                        } else {
                            //
                            val presentationTimeUs = extractor.sampleTime
                            decoder.queueInputBuffer(
                                inputBufIndex,
                                0,
                                chunkSize,
                                presentationTimeUs,
                                0
                            )
                            inputChunk++
                            //移动到下一个样本
                            extractor.advance()
                        }
                    }
                } else {
                    Log.d("hch", "input buffer not available")
                }
            }

            if (!outputDone) {
                //获取Codec的数据
                val decoderStatus = decoder.dequeueOutputBuffer(mBufferInfo, TIMEOUT_USEC)
                when {
                    decoderStatus == MediaCodec.INFO_TRY_AGAIN_LATER -> {
                        Log.d("hch", "no output from decoder available")
                    }
                    decoderStatus == MediaCodec.INFO_OUTPUT_FORMAT_CHANGED -> {
                        val newFormat = decoder.outputFormat
                        Log.d(TAG, "decoder output format changed: $newFormat")
                    }
                    decoderStatus < 0 -> {
                        throw RuntimeException("unexpected result from decoder.dequeueOutputBuffer: $decoderStatus")
                    }
                    else -> {
                        // decoderStatus >= 0
                        var doLoop = false
                        if (mBufferInfo.flags.and(MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                            if (mLoop) {
                                doLoop = true
                            } else {
                                outputDone = true
                            }
                        }
                        val doRender = (mBufferInfo.size != 0)
                        //控制速率
                        if (doRender){
                            callback.preRender(mBufferInfo.presentationTimeUs)
                        }
                        //释放资源
                        decoder.releaseOutputBuffer(decoderStatus,doRender)
                        if (doRender ) {
                            callback.postRender()
                        }
                        if (doLoop){
                            Log.d(TAG,"")
                            extractor.seekTo(0, MediaExtractor.SEEK_TO_CLOSEST_SYNC)
                            inputDone = false
                            decoder.flush()

                        }
                    }
                }

            }

        }
    }

    class PlayTask(
        player: MediaCodecDemo,
    ) :
        Runnable {
        private val mPlayer: MediaCodecDemo
        private var mDoLoop = false
        private var mThread: Thread? = null
        private val mLocalHandler: LocalHandler
        private val mStopLock = java.lang.Object()
        private var mStopped = false


        fun setLoopMode(loopMode: Boolean) {
            mDoLoop = loopMode
        }


        fun execute() {
            mThread = Thread(this, "Movie Player")
            mThread?.start()
        }

        fun waitForStop() {
            synchronized(mStopLock) {
                while (!mStopped) {
                    try {
                        mStopLock.wait()
                    } catch (ie: InterruptedException) {
                        // discard
                    }
                }
            }
        }

        override fun run() {
            try {
                mPlayer.play()
            } catch (ioe: IOException) {
                throw RuntimeException(ioe)
            } finally {
                // tell anybody waiting on us that we're done
                synchronized(mStopLock) {
                    mStopped = true
                    mStopLock.notifyAll()
                }


            }
        }

        private class LocalHandler : Handler() {
            override fun handleMessage(msg: Message) {
                val what = msg.what
                when (what) {
                    MSG_PLAY_STOPPED -> {
                        val fb: PlayerFeedback = msg.obj as PlayerFeedback
                        fb.playbackStopped()
                    }
                    else -> throw RuntimeException("Unknown msg $what")
                }
            }
        }

        companion object {
            private const val MSG_PLAY_STOPPED = 0
        }

        /**
         * Prepares new PlayTask.
         *
         * @param player The player object, configured with control and output.
         * @param feedback UI feedback object.
         */
        init {
            mPlayer = player
            mLocalHandler = LocalHandler()
        }
    }

}

interface PlayerFeedback {
    fun playbackStopped()
}
```



```kotlin
interface FrameCallback {
    fun preRender(presentationTimeUsec: Long)
    fun postRender()
    fun loopReset()
}
```

```kotlin
package com.example.videodemo

import android.util.Log
import com.example.videodemo.SpeedControlCallback

class SpeedControlCallback : FrameCallback {
    private var mPrevPresentUsec: Long = 0
    private var mPrevMonoUsec: Long = 0
    private var mFixedFrameDurationUsec: Long = 0
    private var mLoopReset = false

    fun setFixedPlaybackRate(fps: Int) {
        mFixedFrameDurationUsec = ONE_MILLION / fps
    }

    // runs on decode thread
    override fun preRender(presentationTimeUsec: Long) {

        if (mPrevMonoUsec == 0L) {
            mPrevMonoUsec = System.nanoTime() / 1000
            mPrevPresentUsec = presentationTimeUsec
        } else {
            // Compute the desired time delta between the previous frame and this frame.
            var frameDelta: Long
            if (mLoopReset) {
                mPrevPresentUsec = presentationTimeUsec - ONE_MILLION / 30
                mLoopReset = false
            }
            frameDelta = if (mFixedFrameDurationUsec != 0L) {
              
                mFixedFrameDurationUsec
            } else {
                presentationTimeUsec - mPrevPresentUsec
            }
            if (frameDelta < 0) {
                Log.w(TAG, "Weird, video times went backward")
                frameDelta = 0
            } else if (frameDelta == 0L) {
              
                Log.i(TAG, "Warning: current frame and previous frame had same timestamp")
            } else if (frameDelta > 10 * ONE_MILLION) {
              
                Log.i(
                    TAG, "Inter-frame pause was " + frameDelta / ONE_MILLION +
                            "sec, capping at 5 sec"
                )
                frameDelta = 5 * ONE_MILLION
            }
            val desiredUsec = mPrevMonoUsec + frameDelta // when we want to wake up
            var nowUsec = System.nanoTime() / 1000
            while (nowUsec < desiredUsec - 100 /*&& mState == RUNNING*/) {

                var sleepTimeUsec = desiredUsec - nowUsec
                if (sleepTimeUsec > 500000) {
                    sleepTimeUsec = 500000
                }
                try {
                    if (CHECK_SLEEP_TIME) {
                        val startNsec = System.nanoTime()
                        Thread.sleep(sleepTimeUsec / 1000, (sleepTimeUsec % 1000).toInt() * 1000)
                        val actualSleepNsec = System.nanoTime() - startNsec
                        Log.d(
                            TAG, "sleep=" + sleepTimeUsec + " actual=" + actualSleepNsec / 1000 +
                                    " diff=" + Math.abs(actualSleepNsec / 1000 - sleepTimeUsec) +
                                    " (usec)"
                        )
                    } else {
                        Thread.sleep(sleepTimeUsec / 1000, (sleepTimeUsec % 1000).toInt() * 1000)
                    }
                } catch (ie: InterruptedException) {
                }
                nowUsec = System.nanoTime() / 1000
            }

            mPrevMonoUsec += frameDelta
            mPrevPresentUsec += frameDelta
        }
    }

    override fun postRender() {}
    override fun loopReset() {
        mLoopReset = true
    }

    companion object {
        private const val TAG = "SpeedControlCallback"
        private const val CHECK_SLEEP_TIME = false
        private const val ONE_MILLION = 1000000L
    }
}
```

以上就是mediaCodec的解码mp4视频轨道的代码了。