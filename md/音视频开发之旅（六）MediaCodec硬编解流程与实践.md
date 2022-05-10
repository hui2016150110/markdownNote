# 音视频开发之旅（六）MediaCodec硬编解流程与实践

原创 yabin [音视频开发之旅](javascript:void(0);) *2020-10-24 22:43*

收录于话题

\#音视频之旅65个

\#个人成长75个

![图片](https://mmbiz.qpic.cn/mmbiz_png/4vYVnVshGswLZepUBvO8EeRxHibIhTodcUfLn2j3pfkSjI1rsLx5UzicwP8TR4BRic8NaB0RVQV9kWKSujicxASyAA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 目录 

1. MediaCodec介绍
2. 工作原理和基本流程
3. 数据格式
4. 生命周期
5. 同步和异步模式
6. 流控
7. 实践：AAC解码为PCM同步和异步的两种实现
8. 遇到的问题
9. 参考
10. 收获

## 一、介绍

Android底层多媒体模块采用的是OpenMax框架，实现方都要遵循OpenMax标准。Google默认提供了一系列的软编软解的实现，而硬编硬解则由芯片厂商完成，所以不同芯片的手机，硬编硬解的实现和性能是会有差异的。比如我手机的编解码实现部分如下



![图片](https://mmbiz.qpic.cn/mmbiz/4vYVnVshGswLZepUBvO8EeRxHibIhTodcdBmibnbYmC9rtOibLrwrvmficChpGPruGtSInhllIuOFnP2dNPg212PyA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

MediaCodec提供了一套访问底层多媒体模块的接口供应用层实现编解码功能。

## 二、工作原理和基本流程

MediaCodec使用的基本流程如下：



```
- createByCodeName/createEncoderByType/createDecoderByType: （静态工厂构造MediaCodec对象）--Uninitialized状态
- configure：（配置） -- configure状态
- start        （启动）--进入Running状态
- while(1) {
    try{
       - dequeueInputBuffer    （从编解码器获取输入缓冲区buffer）
       - queueInputBuffer      （buffer被生成方client填满之后提交给编解码器）
       - dequeueOutputBuffer   （从编解码器获取输出缓冲区buffer）
       - releaseOutputBuffer   （消费方client消费之后释放给编解器）
    } catch(Error e){
       - error                   （出现异常 进入error状态）
    }
    
}
- stop                          （编解码完成后，释放codec）
- release
```

基本流程结合代码两张Buffer队列示意图和生命周期图一起看，整个流程就会很清晰。



![图片](https://mmbiz.qpic.cn/mmbiz/4vYVnVshGswLZepUBvO8EeRxHibIhTodc0xyJBHGwncYIib8c9uhsg9ia99qoTDIbLn3CZNICWKmmqvMRmh7eDsug/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

![图片](https://mmbiz.qpic.cn/mmbiz/4vYVnVshGswLZepUBvO8EeRxHibIhTodcOwsjwr49XO5X4iaVqPbLKicibQUHbMBqWicrnyhgmZmObDsjyV00Ls3ovA/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

下面我们重点看下核心的部分Buffer队列的操作。
MediaCodec采用了2个缓冲区队列（inputBuffer和outputBuffer），异步处理数据，



```
1. 数据生成方（左侧Client）从input缓冲队列申请empty buffer—》dequeueinputBuffer
2. 数据生产方（左侧Client）把需要编解码的数据copy到empty buffer，然后繁缛到input缓冲队列 —》queueInputBuffer
3. MediaCodec从input缓冲区队列中取一帧进行编解码处理
4. 编解码处理结束后，MediaCodec将原始inputbuffer置为empty后放回左侧的input缓冲队列，将编解码后的数据放入到右侧output缓冲区队列
5. 消费方Client（右侧Client）从output缓冲区队列申请编解码后的buffer —》dequeueOutputBuffer
6. 消费方client（右侧Client）对编解码后的buffer进行渲染或者播放
7. 渲染/播放完成后，消费方Client再将该buffer放回到output缓冲区队列 —》releaseOutputBuffer
```

## 三、数据格式

Mediacodec接受三种数据格式和两种载体 分别如下：
压缩数据、原始音频数据和原始视频数据
以Surface作为载体或者ByteBuffer作为载体

1. 压缩数据
   压缩数据可以作为解码器的输入、编码器的输出，需要指定数据的格式，这样codec才知道如何处理这些压缩数据
2. 原始音频数据 — PCM音频数据帧
3. 原始视频数据
   视频看解码支持的常用的色彩格式有 native raw video format和 flexible YUV buffers
   native raw video format : COLOR_FormatSurface，可以用来处理surface模式的数据输入输出。
   flexible YUV buffers : 例如COLOR_FormatYUV420Flexible,可以用来处理surface模式的输出输出，在使用ByteBuffer模式的时候可以用getInput/OutputImage(int)方法来获取image数据。

## 四、生命周期

![图片](https://mmbiz.qpic.cn/mmbiz/4vYVnVshGswLZepUBvO8EeRxHibIhTodc0xyJBHGwncYIib8c9uhsg9ia99qoTDIbLn3CZNICWKmmqvMRmh7eDsug/640?wx_fmt=other&wxfrom=5&wx_lazy=1&wx_co=1)

MediaCodec生命周期状态分为三种 Stopped、Executing和Released
在上面第一部分工作原理和基本流程中我们也简单提到了，下面我们详细说下
其中Stopped包含三种子状态 Uninitialized（为初始化状态）、Configured（已配置状态）、Error（异常状态）
Executing也包含三个子状态 Flushed（刷新状态）、Running（运行状态）和EOS（流结束状态）
我们重点看下Executing状态
在调用mediacodec.start()方法后编解码器立即进入Executing状态的Flush子状态，此时编解码器会拥有所有的inputBuffer
一旦第一个输入缓存inputbuffer被移出队列（即：queueInputBuffer），编解码器转为Running状态，编解码器的大部分生命周期会在此状态下。
当带有end-of-stream标记的inputBuffer入队列时（queueInputBuffer），编解码器将转入EOS状态。在这种状态下，编解码器不再接收新的inputBuffer，但是仍然产生outputBuffer，直到end-of-stream标记到达输出端。
可以在Executiong下的任何时候调用flush（）使编解码器重新回到Flushed状态。

## 五、同步异步模式

Buffer的生产消费有种模式，一种是同步模式，即本文第一部分介绍的流程方式。从android5.0 google推出了异步模式，通过给codec设置回调setCalback进行buffer的生产消费操作。
(img)
官方给出的典型代码如下：



```
MediaCodec codec = MediaCodec.createByCodecName(name);
MediaFormat mOutputFormat; // member variable
codec.setCallback(new MediaCodec.Callback() {
   @Override
   void onInputBufferAvailable(MediaCodec mc, int inputBufferId) {
     ByteBuffer inputBuffer = codec.getInputBuffer(inputBufferId);
     // fill inputBuffer with valid data
     …
     codec.queueInputBuffer(inputBufferId, …);
   }

   @Override
   void onOutputBufferAvailable(MediaCodec mc, int outputBufferId, …) {
     ByteBuffer outputBuffer = codec.getOutputBuffer(outputBufferId);
     MediaFormat bufferFormat = codec.getOutputFormat(outputBufferId); // option A
     // bufferFormat is equivalent to mOutputFormat
     // outputBuffer is ready to be processed or rendered.
     …
     codec.releaseOutputBuffer(outputBufferId, …);
   }

   @Override
   void onOutputFormatChanged(MediaCodec mc, MediaFormat format) {
     // Subsequent data will conform to new format.
     // Can ignore if using getOutputFormat(outputBufferId)
     mOutputFormat = format; // option B
   }

   @Override
   void onError(…) {
     …
   }
 });
 codec.configure(format, …);
 mOutputFormat = codec.getOutputFormat(); // option B
 codec.start();
 // wait for processing to complete
 codec.stop();
 codec.release();
```

## 六、MediaCodec 流控

编码器可以设置一个目标码率，但编码器的实际输出码率不会完全符合设置，在编码过程中实际可以控制的并不是最终的输出码率，而是编码过程中的一个量化参数（Quantiaztion Parameter QP）,它和码率并没有固定的关系，而是取决于图像内容。
android码率控制有两种模式
一种是设置cofigure时设定目标码率和码率控制模式，



```
mediaFormat.setInteger(MediaFormat.KEY_BIT_RATE, bitRate);
mediaFormat.setInteger(MediaFormat.KEY_BITRATE_MODE,
MediaCodecInfo.EncoderCapabilities.BITRATE_MODE_VBR);
mVideoCodec.configure(mediaFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
```

码率控制模式有三种：
CQ 表示完全不控制码率，尽最大可能保证图像质量, 质量要求高、不在乎带宽、解码器支持码率剧烈波动的情况下，可以选择这种策略；
CBR 表示编码器会尽量把输出码率控制为设定值，输出码率会在一定范围内波动，对于小幅晃动，方块效应会有所改善，但对剧烈晃动仍无能为力；连续调低码率则会导致码率急剧下降，如果无法接受这个问题，那 VBR 就不是好的选择；
VBR 表示编码器会根据图像内容的复杂度（实际上是帧间变化量的大小）来动态调整输出码率，图像复杂则码率高，图像简单则码率低，优点是稳定可控，这样对实时性的保证有帮助。所以 WebRTC 开发中一般使用的是CBR；

另一种是动态的调整目标码率。



```
Bundle param = new Bundle();
param.putInt(MediaCodec.PARAMETER_KEY_VIDEO_BITRATE, bitrate);
mediaCodec.setParameters(param);
```

## 七、实践：AAC解码为PCM同步和异步的两种实现 （音频解码）

目的：通过该功能的实践，熟悉mediaCodec的流程，以及同步和异步两种方式实现
具体实现如下：



```
package com.av.mediajourney.mediacodec;

import android.content.Context;
import android.media.MediaCodec;
import android.media.MediaExtractor;
import android.media.MediaFormat;
import android.os.Build;
import android.os.Environment;
import android.text.TextUtils;
import android.util.Log;
import android.widget.Toast;

import androidx.annotation.NonNull;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;

public class AACToPCMDelegate {

    private static final String TAG = "AACToPCMDelegate";
    private static final long TIMEOUT_US = 0;
    private Context mContext;

    private ByteBuffer[] inputBuffers;
    private ByteBuffer[] outputBuffers;
    private MediaExtractor mediaExtractor;
    private MediaCodec decodec;

    private FileOutputStream fileOutputStream;

    public AACToPCMDelegate(MediaCodecActivity context) {
        this.mContext = context;
    }

    /**
     * 通过aacToPCM 熟悉mediaCodec的流程，以及通过同步和异步两种方式实现
     */
    void aacToPCM() {
        boolean isAsync = true;
        if (isAsync && Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
            isAsync = false;
        }

        //1. initFile
        File file = initFile(isAsync);
        if (file == null) {
            return;
        }

        //2. 初始化mediaCodec
        initMediaCodec(file.getAbsolutePath(), isAsync);
        if (decodec == null) {
            Toast.makeText(mContext, "decodec is null", Toast.LENGTH_SHORT).show();
            return;
        }

        if (!isAsync) {
            //同步处理
            decodecAacToPCMSync();
        }
    }

    private File initFile(boolean isAsync) {
        String child = isAsync ? "aacToPcmAsync.pcm" : "aacToPcmSync.pcm";
        File outputfile = new File(mContext.getExternalFilesDir(Environment.DIRECTORY_MUSIC), child);
        if (outputfile.exists()) {
            outputfile.delete();
        }
        try {
            fileOutputStream = new FileOutputStream(outputfile.getAbsolutePath());
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
        File file = new File(mContext.getExternalFilesDir(Environment.DIRECTORY_MUSIC), "sanguo.aac");
        if (!file.exists()) {
            Toast.makeText(mContext, "文件不存在", Toast.LENGTH_SHORT).show();
            return null;
        }
        return file;
    }

    private void initMediaCodec(String path, boolean isASync) {
        mediaExtractor = new MediaExtractor();
        try {
            mediaExtractor.setDataSource(path);
            int trackCount = mediaExtractor.getTrackCount();
            for (int i = 0; i < trackCount; i++) {

                MediaFormat trackFormat = mediaExtractor.getTrackFormat(i);
                String mime = trackFormat.getString(MediaFormat.KEY_MIME);
                if (TextUtils.isEmpty(mime)) {
                    continue;
                }
                Log.i(TAG, "initMediaCodec: mime=" + mime);
                if (mime.startsWith("audio/")) {
                    mediaExtractor.selectTrack(i);
                }
                //生成MediaCodec，此时处于Uninitialized状态
                decodec = MediaCodec.createDecoderByType(mime);
                //configure 处于Configured状态
                decodec.configure(trackFormat, null, null, 0);

                if (isASync) {
                    setAsyncCallBack();
                }
                //处于Excuting状态 Flushed子状态
                decodec.start();
                if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
                    inputBuffers = decodec.getInputBuffers();
                    outputBuffers = decodec.getOutputBuffers();
                    Log.i(TAG, "initMediaCodec: inputBuffersSize=" + inputBuffers.length + " outputBuffersSize=" + outputBuffers.length);
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    /**
     * 成功输出到目标文件
     * 到处生成的pcm，用ffplay播放pcm文件 发现和之前的aac是一样的
     * ffplay -ar 44100 -channels 2 -f s16le -i /Users/yabin/Desktop/tmp/aacToPcm.pcm
     */
    private void decodecAacToPCMSync() {
        boolean isInputBufferEOS = false;
        boolean isOutPutBufferEOS = false;

        while (!isOutPutBufferEOS) {
            if (!isInputBufferEOS) {
                //1. 从codecInputBuffer中拿到empty input buffer的index
                int index = decodec.dequeueInputBuffer(TIMEOUT_US);
                if (index >= 0) {
                    ByteBuffer inputBuffer;
                    //2. 通过index获取到inputBuffer
                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                        inputBuffer = decodec.getInputBuffer(index);
                    } else {
                        inputBuffer = inputBuffers[index];
                    }
                    if (inputBuffer != null) {
                        Log.i(TAG, "decodecAacToPCMSync: " + "  index=" + index);

                        inputBuffer.clear();
                    }
                    //extractor读取sampleData
                    int sampleSize = mediaExtractor.readSampleData(inputBuffer, 0);
                    //3. 如果读取不到数据，则认为是EOS。把数据生产端的buffer 送回到code的inputbuffer
                    Log.i(TAG, "decodecAacToPCMSync: sampleSize=" + sampleSize);
                    if (sampleSize < 0) {
                        isInputBufferEOS = true;
                        decodec.queueInputBuffer(index, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM);
                    } else {
                        decodec.queueInputBuffer(index, 0, sampleSize, mediaExtractor.getSampleTime(), 0);
                        //读取下一帧
                        mediaExtractor.advance();
                    }
                }
            }

            MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
            //4. 数据消费端Client 拿到一个有数据的outputbuffer的index
            int index = decodec.dequeueOutputBuffer(bufferInfo, TIMEOUT_US);
            if (index < 0) {
                continue;
            }
            ByteBuffer outputBuffer;
            //5. 通过index获取到inputBuffer
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                outputBuffer = decodec.getOutputBuffer(index);
            } else {
                outputBuffer = outputBuffers[index];
            }

            Log.i(TAG, "decodecAacToPCMSync: outputbuffer index=" + index + " size=" + bufferInfo.size + " flags=" + bufferInfo.flags);
            //把数据写入到FileOutputStream
            byte[] bytes = new byte[bufferInfo.size];
            outputBuffer.get(bytes);
            try {
                fileOutputStream.write(bytes);
                fileOutputStream.flush();
            } catch (IOException e) {
                e.printStackTrace();
            }

            //6. 然后清空outputbuffer，再释放给codec的outputbuffer
            decodec.releaseOutputBuffer(index, false);
            if ((bufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                isOutPutBufferEOS = true;
            }
        }

        close();
    }

    private void close() {
        mediaExtractor.release();
        decodec.stop();
        decodec.release();
        try {
            fileOutputStream.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void setAsyncCallBack() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            decodec.setCallback(new MediaCodec.Callback() {
                @Override
                public void onInputBufferAvailable(@NonNull MediaCodec codec, int index) {
                    Log.i(TAG, "setAsyncCallBack - onInputBufferAvailable: index=" + index);
                    //1. 从codecInputBuffer中拿到empty input buffer的index
                    if (index >= 0) {
                        ByteBuffer inputBuffer;
                        //2. 通过index获取到inputBuffer
                        inputBuffer = decodec.getInputBuffer(index);
                        if (inputBuffer != null) {
                            Log.i(TAG, "setAsyncCallBack- onInputBufferAvailable: " + "  index=" + index);
                            inputBuffer.clear();
                        }
                        //extractor读取sampleData
                        int sampleSize = mediaExtractor.readSampleData(inputBuffer, 0);
                        //3. 如果读取不到数据，则认为是EOS。把数据生产端的buffer 送回到code的inputbuffer
                        Log.i(TAG, "setAsyncCallBack- onInputBufferAvailable: sampleSize=" + sampleSize);
                        if (sampleSize < 0) {
                            decodec.queueInputBuffer(index, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM);
                        } else {
                            decodec.queueInputBuffer(index, 0, sampleSize, mediaExtractor.getSampleTime(), 0);
                            //读取下一帧
                            mediaExtractor.advance();
                        }
                    }
                }

                @Override
                public void onOutputBufferAvailable(@NonNull MediaCodec codec, int index, @NonNull MediaCodec.BufferInfo bufferInfo) {
                    Log.i(TAG, "setAsyncCallBack - onOutputBufferAvailable: index=" + index + " size=" + bufferInfo.size + " flags=" + bufferInfo.flags);
                    //4. 数据消费端Client 拿到一个有数据的outputbuffer的index
                    if (index >= 0) {
                        ByteBuffer outputBuffer;
                        //5. 通过index获取到inputBuffer
                        outputBuffer = decodec.getOutputBuffer(index);

                        Log.i(TAG, "setAsyncCallBack - onOutputBufferAvailable: outputbuffer index=" + index + " size=" + bufferInfo.size + " flags=" + bufferInfo.flags);
                        //把数据写入到FileOutputStream
                        byte[] bytes = new byte[bufferInfo.size];
                        outputBuffer.get(bytes);
                        try {
                            fileOutputStream.write(bytes);
                            fileOutputStream.flush();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }

                        //6. 然后清空outputbuffer，再释放给codec的outputbuffer
                        decodec.releaseOutputBuffer(index, false);
                        if ((bufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
                            close();
                        }
                    }

                }

                @Override
                public void onError(@NonNull MediaCodec codec, @NonNull MediaCodec.CodecException e) {
                    Log.e(TAG, "setAsyncCallBack - onError: ");
                }

                @Override
                public void onOutputFormatChanged(@NonNull MediaCodec codec, @NonNull MediaFormat format) {
                    Log.i(TAG, "setAsyncCallBack - onOutputFormatChanged: ");

                }
            });
        }

    }

}
```

## 八、遇到的问题



```
java.lang.IllegalStateException
        at android.media.MediaCodec.getBuffer(Native Method)
        at android.media.MediaCodec.getInputBuffer(MediaCodec.java:3246)
 
int index = decodec.dequeueInputBuffer(TIMEOUT_US); _
之后直接进行了inputbuffer的处理，而没有判断index是否有效（index>=0）
```

## 九、参考

1. MeidaCodec官方文档
2. Android音频开发（5）：音频数据的编解码
3. MediaCodec进行AAC编解码（文件格式转换）
4. Android MediaCodec stuff

## 十、收获

强烈建议从示例代码开始了解MediaCodec，而不是试图从文档把它搞清楚。

1. 了解了MediaCodec工作原理和基本流程
2. 生命周期的应用了解
3. codec的同步和异步的使用和场景
4. 流控的设置
5. 通过AAC解码为PCM，同步和异步两种实现逐步理解原理流程
6. 遇到的问题总结复盘

感谢你的阅读。
下一篇我们接学习实践SurfaceView 、GLSurfaceView、TextureView 、SurfaceTexture、Surface,以及它们的关系，使用方法场景和优缺点。

欢迎关注“音视频开发之旅”，你的“在看”、“点赞”、“分享”都是莫大的支持。我们下篇见。

音视频开发之旅 发起了一个读者讨论欢迎讨论讨论已结束



收录于话题 #音视频之旅

 65个

上一篇音视频开发之旅（七） OpenGL ES 基本概念下一篇音视频开发之旅（五）MediaExtractor MediaMuxer 实现视频的解封装与合成

阅读 230

分享收藏

62