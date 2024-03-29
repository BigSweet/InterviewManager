![image-20200717182032542](../image/image-20200717182032542.png)

## 日志

```
导入日志库
include <libavutil/log.h>
设置日志级别
av_log_set_level(AV_LOG_DEBUG)
打印日志
av_log(NULL,AV_LOG_INFO,"...%s\n",op)
```



## 上下文概念

```
AVFormatContext //流媒体上下文。用来和多媒体文件建立连接
AVStream //多媒体文件中的流媒体
AVPacket //流媒体中的包,发送给解码器解码
```



## 媒体信息相关

```
av_register_all() //注册编解码库
avformat_open_input() //打开流媒体文件
avformat_close_input() //释放资源
av_dump_format()  //打印流媒体信息

```

example

```c++
 av_register_all(); //注册服务
    char *mediaUrl = "/Users/yanzhe/ffmpeg/ffmpeg_install/ffmpeg/bin/input.mp4";
    AVFormatContext *avFormatContext = avformat_alloc_context();
    int openRet = avformat_open_input(&avFormatContext, mediaUrl, NULL, NULL);
    if (openRet < 0) {
        av_log(NULL, AV_LOG_INFO, "%s\n", "打开文件失败");
    }
    av_dump_format(avFormatContext, 0, mediaUrl, 0);//输入流还是输出流 这是输入流填0输出流填1
    avformat_close_input(&avFormatContext);

```

## 抽取音视频

```
av_init_packet()//初始化结构体 
av_find_best_stream()//找到最好的流媒体
av_read_frame()//读取流中的数据包
av_packet_unref() //释放引用计数，释放包
抽取的音频需要添加adts头才能播放
抽取的视频需要在普通帧添加标识码，关键帧前后添加sps,pps才能播放
```



## 音视频格式转换

```

 avformat_alloc_output_context2() //分配输出上下文 
    avformat_free_context() //释放输出上下文
    avformat_new_stream() //给媒体文件添加新的流
    avcodec_parameters_copy() //将输入的流媒体codecpar复制到新的媒体流中
    avformat_write_header() //分配一个流数据并且写入头部
    进行时间基转换
    av_interleaved_write_frame() //将帧数据写入输入文件中
    av_write_trailer()
```

