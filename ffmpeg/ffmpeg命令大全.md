![image-20200716144226821](/Users/yanzhe/android/知识整理/image/image-20200716144226821.png)



## 录制相关

```
ffmpeg -f avfoundation -list_devices true -i ""//查看设备索引

ffmpeg -f avfoundation -i 1 -r 30 out.yuv  //录制屏幕

-f 指定采集视频用到的库

-i 采集桌面

-r 帧率

ffplay -s 2880x1800 -pix_fmt uyvy422 out.yuv//播放这个格式的视频 
-s 指定大小
-fix_fmt 指定格式


ffmpeg -f avfoundation -i :0 out.wav //录制音频
-i 采集哪个设备的数据
:前面是指定视频 后面是指定音频

ffplay out.wav//播放音频

ffmpeg -f avfoundation -r 30 -i 0:0 out.yuv //录制音频和视频
```

## 转换相关

```
ffmpeg -i outt.flv -vcodec copy -acodec copy outt.mp4 转换格式

ffmpeg -i input.mp4 -acodec copy -vn out.aac 抽取视频中的音频流

ffmpeg -i input.mp4 -vcodec copy -an out.h264 抽取视频流为H264或者MP4
```



## 处理原始数据相关



```
ffmpeg -i input.mp4 -an -c:v rawvideo -pix_fmt yuv420p out.yuv
//抽取视频中的原始数据yuv420p
可以通过ffmpeg -pix_fmt 查看当前支持的格式
ffplay -s 852x480 out.yuv //播放这个视频

ffmpeg -i input.mp4 -vn -ar 44100 -ac 2 -f s16le out.pcm
-ar audioration 音频采样率
-ac2 指定双声道
-f 数据存储格式 有符号16小头

//播放音频
ffplay -ar 44100 -ac 2 -f s16le out.pcm
```



## 裁剪和合并

```
ffmpeg -i out.h264 -i out.aac -vcodec copy -acodec copy outt.mp4 合并音频流和视频流
ffmpeg -i input.mp4 -ss 00:00:00 -t 10 out.mp4 //裁剪视频 取input.mp4的前面10秒
ffmpeg -f concat -i inputs.txt out.flv //识别inputs中的视频文件合并到一起生成out.flv
input.txt中的格式为
file '1.ts'
file '2.ts'
格式需要支持拼接的才行
```



## 滤镜相关

```
解码数据后，进行过滤，在进行编码，在进行输出
//宽高裁剪  
ffmpeg -i input.mp4 -vf crop=in_w-200:in_h-200 -c:v libx264 -c:a copy out.mp4
//裁剪滤镜 裁剪为宽高200 200 用libx264编码器 音频不处理

ffmpeg -i input.mp4 -i logo1.png -filter_complex overlay=W-w:56 -max_muxing_queue_size 1024 ring_logo_t.mp4
//添加水印
overlay=x:y
X方向的距离W-w
W为视频宽度w为图片宽度
Y方向距离56
```



## 图片和视频互转命令

```
 ffmpeg -i input.mp4 -r 1 -f image2 image-%3d.jpeg
 //指定input视频转为图片
 -r 每秒转出几张图片
 -f 指定输出图片文件格式
 image-%3d 输出文件的文件名字 image开头后面是3个数字比如001
 
 ffmpeg -i image-%3d.jpeg outss.mp4 //图片转视频
```



## 直播推拉流

```
ffmpeg -re -i out.mp4 -c copy -f flv rtmp:://server/live/starmName //推流
-c 音视频编解码
-f 推出去的文件的格式
-re 减慢帧率速度 保持帧率同步

ffmpeg -i rtmp:://server/live/starmName -c copy dump.flv //拉流
拉流输出为什么格式的文件
```

