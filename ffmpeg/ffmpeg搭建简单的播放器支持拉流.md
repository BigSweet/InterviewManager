### Mac下编译ffmpeg

```
git clone git@github.com:FFmpeg/FFmpeg.git
//生成makefile文件

./configure --prefix=/usr/local/ffmpeg --enable-gpl --enable-nonfree --enable-libfdk-aac --enable-libx264 --enable-libx265 --enable-filter=delogo --enable-debug --disable-optimizations --enable-libspeex --enable-videotoolbox --enable-shared --enable-pthreads --enable-version3 --enable-hardcoded-tables --disable-x86asm --cc=clang --host-cflags= --host-ldflags=
```

```
报错libfdk_aac not found
brew install fdk-aac

报错Your Xcode (1) is too outdated.
Please update to Xcode 8.3.2 (or delete it).
Xcode can be updated from

更新brew 
brew update-reset

继续报错
Error: The following directories are not writable by your user:
/usr/local/share/man/man3
/usr/local/share/man/man5
/usr/local/share/man/man7
执行sudo chown -R $(whoami) /usr/local/share/man/man3 /usr/local/share/man/man5 /usr/local/share

brew install fdk-aac 成功

报错 speex not found using pkg-config
brew install speex
brew install pkg-config

报错264 265找不到
brew install x264
brew install x265
brew install sdl2  // 没有安装sdl2无法安装ffplay

最后生成一个config文件，在makefile中被引用ffbuild/config.mak
执行make && make install
会在/usr/local/ffmpeg 目录生成库文件
```















## 搭建播放器



4个重要的类  
AVPacket      解码前的数据结构体
AvFrame       解码后的数据结构体
AVFormatContext  媒体文件的构成和基本上下文
AVCodecContext 解码信息上下文

首先是初始化网络，在获取avformatcontext，这个东西可以封装媒体的信息
通过这个context去打开java层传递过来的path，获取里面的流媒体信息，一般有音频流和视频流，
这个流媒体信息里面会有一个codeid，表示的是他们的流媒体的格式，通过这个格式在avcodec库中寻找解码器
找到解码器之后，把流媒体信息的数据，发送给这个解码器，进行解码。音视频都是一样的
视频通过java层传递过来的surfaceview转换为C++层的nativewindows进行播放
音频通过ndk中的opensles引擎来进行播放

FFmpeg搭建播放器的关键流程为

第一步开启网络对应代码为
avformat_network_deinit();
获取avformatcontext
avFormatContext = avformat_alloc_context();
打开网络
avformat_open_input(&avFormatContext, path, 0, 0)
获取媒体流
avformat_find_stream_info(avFormatContext, 0)
在avFormatContext->nb_streams中会有音频流和视频流
流中会有格式信息
根据格式信息寻找相对应的解码器
avcodec_find_decoder(parameters->codec_id)

视频解码
读取avformatcontext中的帧数据
av_read_frame放入一个队列中开启线程取队列中的数据发送给解码器进行解码
avcodec_send_packet  发送解码数据
avcodec_receive_frame 获取解码数据
将解码好的数据放入另外一个队列中
从解码好的队列中取出数据，通过sws_scale转换之后，开始播放视频
java层传递一个surfaceview到C++层
通过surfaceview获取C++层的ANativeWindow开始渲染画面

音频解码
swr_convert
通过avformatcontext获取数据，发送给解码器，放入队列中，在将解码后的数据放入另外一个队列
从这个队列中取出数据
通过opensles引擎播放音频
流程为
创建引擎engineObject
初始化
获取引擎接口engineInterface
通过引擎接口创建混音器
初始化混音器outputMixObject
开始播放

音视频同步

一般是音频像视频同步，因为视频有关键帧需要依赖下一帧才能解析出来。这个时候音频进行休眠等待，进行同步





