title: 音视频编码基础和ffmpeg
date: 2018-11-05 19:11:12
tags:
	- ffmpeg
categories: tech
comments: true
---
雷霄骅csdn课程学习的浓缩总结,去除了编码细节，只强调主要概念和重点工具。[学习地址](https://blog.csdn.net/leixiaohua1020/article/details/47068015)
##音视频播放流程
![视频播放流程](a.png)

### 主要概念
1. 封装格式（AVI,mp4,TS,FLV,MKV,RMVB） ;
2. 视频编码格式(HEVC、H.264、MPEG4、MEPEG2、VP9、VP8、VC-1);
3. 音频编码格式（AAC，MP3等）;
4. 视频像素数据(RGB24、RGB32、YUV420P、YUV422P、YUV444P；实际视频都是用的YUV);
5. 音频采样数据（PCM无损数据）。

### 码率
编码后一秒的数据量，低就不清晰，高就会质量好。

###H.264
1. 序列>图像(帧)>片组>片>NALU>宏块>亚宏块>块像素
2. DCT->量化->帧内预测->帧间预测->熵编码->环路滤波等
3. 图像数据压缩100倍以上

### AAC ：主流音频处理
将PCM采样音频数据压缩10倍以上。

### RGB24大小示例
一小时体积3600*25*1920*1080*3 = 559.9GByte
频率家定位25Hz，采样精度为8bit，即1个点就是8比特的数据。
* BMP就是存储的RGB格式的像素数据。

### YUV
先存整张图片的Y信息，再存U的信息，再存V的信息，所以读取可以按一张图一张图去读
常用YUV420数据格式
4:4:4 表示4个像素点采样4个Y信息，4个Cb和4个Cr，所以整体像素点单位的3倍
4:2:2 代表4个像素点采样4个Y信息，2个Cb和2个Cr
4:2:0 表示4个像素点采样4个Y信息，横纵向各只有一个Cb/Cr，所以整体像素点单位的1.5倍，一般两个Y共用相邻的Cb和Cr。

###PCM大小示例
4分钟的PCM
4*60*44100*2（双声道）*2（采样精度） = 42.3MByte
采样率是44100HZ常规参数，采样精度为16bit一个


### PCM
如果是单声道，依次存储就好了；如果是双声道，左右声道交错存储

## 工具类
* MediaInfo可以用来查看视频综合信息
* YUV Player可以查看视频像素数据（没有头文件，需要自己打开后在参数中指定高宽和YUV格式）
* 视频播放器vlc、movist
* 封装工具格式查看器 Elecard Format Analyzer
* 视频编码格式查看器 Elecard Stream Eye

![视频编码格式查看](b.png)

* 音频采样数据查看工具：Adobe Audition（没有头文件，需要自己打开后在参数中指定采样率、声道和采样精度）

## ffmpeg命令行学习
* [官方文档](https://www.ffmpeg.org/ffmpeg.html)
* 样例比如ffmpeg -i input.mp4 -i logo.png -filter_complex "[1:v]scale=50:50[a];[0:v][a]overlay=0:0" output.mp4
将图片放到视频每一帧的左上角；
* ffmpeg -i stream1 -i stream2 -i zbxh1.png -filter_complex "[1:v][2:v]overlay=W-w:H-h[a];[0:v]drawbox=0:0:240:162:black@0.5[b];[b][a]overlay=0:0" -acodec aac -ac 1 -vcodec libx264 -deblock 0 -f flv rtmp://localhost:1935/myapp/test 把输入的视频文件换成流地址（亲测过http,rtmp,rtsp等的同种叠加和混搭），把保存的文件换成推送的地址，最后指明编码器等参数。
其中-deblock 0 选项是“去块效应”的，以免视频编码完成后产生块效应。

### ffplay
简单播放工具，可以学习下[官方文档](https://www.ffmpeg.org/ffplay.html)

###ffmpeg关键类库（八大金刚）
* avcodec ：编解码
* avformat：封装格式处理
* avfilter：滤镜特效处理
* avdevice：各种设备输入输出；
* avutil：工具库（大部分库都需要这个库支持）
* postproc：后加工
* swresample：音频采样数据格式转换
* swscale： 视频像素数据格式转换；
###ffmpeg解码数据结构
![解码数据结构](c.png)


* AVFormatContext 最外层封装结构上下文；
	* iformat 输入视频的 AVInputFormat
	* nb_streams 流数量
	* streams 流的数组
	* duration：时长
	* bit_rate ：码率
* AVInputFormat指明封装格式
	* name 封装格式名称
	* long_name:封装格式的长名称
	* extensions：封装格式扩展名
	* id：封装格式ID 
* AVStream主要是视频流和音频流，0一般是视频流，1为音频流
	* id序号
	* codec：流对应 AVCodecContext
	* time_base:该流的时基础，记录时间播放的基数，比如1s
	* r_frame_rate：1s有多少画面
* AVCodecContext 处理编解码上下文
	* codec：编解码的 	AVCodec
	* width、height
	* pix_fmt：像素格式（针对视频）
* AVCodec 解码格式解码器
	* name
	* long_name
	* type
	* id 	

#### 基础数据结构
AVPacket(解码前压缩数据)->AVFrame（解码后的数据）
##### AVPacket 
* pts：显示时间戳，只能是整数，单位是对应的时间基数AVStream的time_base；
* dts：解码时间戳（解码顺序和显示顺序不一样）
* data：压缩编码数据
* size：压缩编码数据大小
* stream_index:所属的AVStream

##### AVFrame
* data：解码后的图像数据（音频采样数据）
* linesize：对视频是图像一行像素大小，对音频是整个音频帧大小
* width、height（专用于视频）
* key_frame：是否是关键帧（只对视频有用）
* pict_type:帧类型（I、P、B专用于视频）


##### sws_scale
这是解码最后的一步，解码出来之后不是最终的分辨率，比如原本480X320，输出520X480，需要进行裁剪去除黑边。

## SDL（Simple DirectMedia Layer）视频显示
* 封装复杂音视频底层交互，简化音视频处理难度。
* 跨平台、开源
* 主要用于做游戏的
SDL架构图
![SDL架构图](d.png)
SDL流程图
![SDL流程图](e.png)

###核心结构
![SDL视频数据结构图](f.png)


