title: 《ffmpeg从入门到精通》关键点
date: 2018-12-01 19:11:12
tags:
	- ffmpeg
categories: tech
comments: true
---
###ffmpeg学习(学习时是4.1版本)
豆瓣阅读购买电子书（ffmpeg从入门到精通 https://read.douban.com/ebook/49786757/）
####简介（Fast Forward 和 Moving Picture Experts Group）
1. 音视频编解码工具和编解码开发套件；
2. 提供多种媒体格式的封装和解封装，包括多种音视频编码，多种协议流媒体、多种色彩格式转换、多种采样率转换、多种码率转换等，提供了丰富的插件模式，包含封装、解封装、编码解码插件。
3. 出名的开源社区引用：ijkplayer、ffmpeg2theora、VLC、MPlayer等，是在 LGPL/GPL 协议 下发布的(如果使用了 GPL 协议发布的模块则必须采用 GPL 协议)，任何人都可以自由使 用，但必须严格遵守 LGPL/GPL 协议 。

####组成
1. 包含AVFormat、 AVCodec、 AVFilter、 AVDevice、 AVUtil 等 模块库；
	* AVFormat 用于媒体封装格式处理，包括文件MP4、FLV等格式和RTMP等网络协议封装格式，可以自定义对应的封装格式支持能力；
	* AVCodec 用于编解码格式支持处理，默认支持MPEG4、AAC等自带媒体编码，还支持第三方的编解码器，比如H.264用x264编码器，H.265用x265；
	* AVFilter 用于通用的音视频、字幕等滤镜处理框架;
	* swscale主要用于高级别的图像转换api，常用于视频从1080p转换为720P等缩放；
	* swresample 音频重采样API，操作音频采样、音频通道布局转换和布局调整。
2. ffplay用来播放，系统需要有SDL来支撑播放，ffplay提供了音视频显示和播放相关的图像信息，音频的波形信息等。
3. ffprobe用于分析多媒体文件，从中获取你想要了解的媒体信息。
4. ffmpeg源码中可以用：
	* configure --help查看所需要的第三方外部库和对应的支持编译参数；
	* configure --list-encoders、configure --list-decoders可以用来查看编译支持的编解码器；
	* configure --list--muxers、configure --list--demuxers可以用来查看支持的封装和解封装 容器格式；
	* configure --list-protocols可以查看支持的流媒体协议。
	*  ./configure  --prefix=/usr/local --enable-gpl --enable-libx264 --enable-libass --enable-ffplay  --extra-cflags="-Os -fpic -I/Users/frostpeng/privateWorkspace/ffmpeg/x264/include $ADDI_CFLAGS" --extra-ldflags="-L/Users/frostpeng/privateWorkspace/ffmpeg/x264/lib $ADDI_LDFLAGS " 支持h264和ffplay编译ffmpeg--enable-libass用来支持字幕
	
5. ffmpeg --help可以查看常规的命令，包括公共操作参数部分、文件主要操作参数、视频操作参数、音频操作参数和字母操作参数;
	* ffmpeg --help看的是基础信息;
	* ffmpeg --help long 可以获得高级参数部分;
	* ffmpeg --help full 可以获得全部帮助信息;
6. ffmpeg常用命令
	* ffmpeg-formats查看支持的视频文件格式;
	* ffmpeg -version查看库版本;
	* ffmpeg -L查看协议;
	* ffmpeg -codecs查看编解码支持;
	* ffmpeg -encoders和-decoders查看编解码器支持;
	* ffmpeg -filters可以查看支持的滤镜；
	* ffmpeg -h 支持查看具体demuxer、muxer、encoder、decoder等操作参数（比如ffmpeg-h muxer=flv）；
	* ffmpeg -h filter=colorkey 、ffmpeg-h decoder=h264 等方式查看详细参数
	* 非常有用比如ffmpeg -h encoder=libx264 可以查看libx264在ffmpeg编码中支持的参数调整；
	* H264的解码可以分为常规、多线程（多线程可以分为帧级别多线程和slice级别多线程）；
	* ffmpeg --help full的AVFormatContext参数部分，用于说明封装转换可使用的参数；
	* ffmpeg --help full的AVCodecContext参数部分，用于说明编解码可使用的参数；
7. ffprobe常用命令
	* ffprobe--help可以查看详细帮助信息；
	* ffprobe ffmpeg yuvviewer streameye mp4info 都可以用来查看媒体信息
	* ffprobe -show_packets input.flv 查看多媒体数据包信息，一般显示很长段的数据
	* ffprobe -show_data -show_packets input.flv 组合查看包的具体数据，一般显示很长段的数据
	* ffprobe -show_format input.flv 可以查看多媒体封装格式
	* ffprobe -show_frames input.flv 查看视频文件具体帧信息
	* ffprobe -show_streams input.flv 查看视频文件流信息
	* ffprobe支持key-value格式的显示方式，如果想要格式化的现实，可以用ffprobe -print_format 或者ffprobe -of参数来进行对应输出，print_format支持多种格式包括xml、json、ini、csv（可以用作Excel展示）、flat等，比如ffprobe -of xml -show_streams input.flv   ；ffprobe -of json -show_streams input.flv;
	* ffprobe -show_frames -select_streams v -of xml input.mp4 可以用select_streams选择展示的流，v视频/a音频/s字幕
8. ffplay常用命令
	* ffplay --help查看全部参数
	* ffplay -ss 30 -t 10 input.mp4 播放ss设置起点，t表示播放多久，相当于播放30s开始，播放10s文件；
	* ffplay -window_title "Hello World, This is a sample" output.mp4 设置播放器标题
	* ffplay -window_title "播放测试"  rtmp://up.v.test.com/live/stream 可以打开网络直播流
	* time ffplay -window_title "Hello World" -ss 20 -t 10 -autoexit output.mp4 通过time查看命令运行时长
	* ffplay -window_title "Test Movie" -vf "subtitles=input.srt" output.mp4 通过filter指定字幕
	* ffplay -showmode 1 output.mp3 可视化查看音频波形
	* ffplay -flags2 +export_mvs -ss 40 output.mp4 -vf codecview=mv=pf+bf+bb 可以查看运动估计显示的图形

#### ffmpeg转封装
1. mp4的格式信息，主要理解Box和FullBox的概念。moov视频头放在mdat前后都可以，但是网络视频为了尽快播放，一般放在前面。
2. mp4分析工具Elecard StreamEye、mp4box、mp4info等
3. ./ffmpeg -i input.flv -c copy -f mp4 -movflags faststart output.mp4 正常情况下ffmpeg生成moov是在mdat写完成之后再写入，可以通过参数faststart将moov容器移动至mdat的前面；
4. FLV也是一种常见格式，注意FLVTAG的概念，主要分为FLV文件头和文件内容。FLV有自己支持的视频编码和音频编码列表，不支持的编码封装时会报错。
5. M3U8 是以文件列表形式存在的支持直播、点播的流媒体格式；

#### ffmpeg软硬件编码
目前已经在学习，主要也是各平台的H264编解码相关内容，没啥大的需要查漏补缺的。gpu编码优化这方面可以考虑在工作中应用。

#### ffmpeg 流媒体

* 目前直播大多数是RTMP、HTTP+FLV、HLS、DASH等方式；
* 包含RTMP、HTTP、RTSP等流媒体协议分析；
* 了解一次编码、多路输出的操作方式，一次视频推多路直播平台；
* 介绍HDS和DASH切片方式的直播支持；
* RTMP常用于实时直播。
* RTSP曾主要用于直播，如今广泛用于安防。

#####RTSP协议细节
* 可以通过TCP 、UDP、HTTP隧道实现。
* UDP容易出现拉流丢包异常，在实时性和可靠性适中时，可以考虑采用TCP方式拉流；
* UDP容易丢包导致花屏、绿屏、灰屏、马赛克等问题；

#####HTTP细节
* 直播和点播都可以用HTTP，ffmpeg就可以用做播放器，也可以用来当服务器；
* http拉流录制和转封装，http传输的flv可以录制为HLS（M3U8）、MP4、FLV等。

##### UDP和TCP流媒体
* 常用于裸传输场景

##### 推流和拉流
* 拉流表示从服务器获取视频数据到客户端播放；
* 推流表示从主播端将视频上传到服务器；


##### ffmpeg推多路流
* 使用管道方式输出多路流和使用tee进行多路流输出；
* 使用管道方式推多路流可以一次编码，多次输出支持所得到的流；
* 使用tee方式

 		./ffmpeg -re -i input.mp4 -vcodec libx264 -acodec aac -f flv "tee:rtmp://publish.chinaffmpeg.com/live/stream1|rtmp://publish.chinaffmpeg.com/live/stream2"
* 一次编码，输出两个字链接的tee协议，输出两路RTMP流。

##### ffmpeg 生成HDS流
* ffmpeg支持文件列表方式的切片直播、点播流，除了HLS外，还支持HDS流切片格式。

##### ffmpeg 生成DASH
* 也是列表方式的直播。

#### ffmpeg 滤镜使用
FFmpeg功能强大的主要原因是其包含了滤镜处理avfilter，FFmpeg的avfilter能够实现的音频、视频、字幕渲染效果数不胜数，并且时至今日还在不断地增加新的功能，除了本章介绍的内容之外，还可以从FFmpeg官方网站的文档页面获得FFmpeg的avfilter更多的信息。

#### ffmpeg 采集设备
我们可以了解到Linux、OS X、Windows上的设备采集方式，内容涉及fbdev、v4l2、x11grab、avfoundation、dshow、vfwcap、gdigrab等。

###总评
除了熟悉下命令行，其余价值很低，配不上从入门到精通这种名字，因为没法精通



