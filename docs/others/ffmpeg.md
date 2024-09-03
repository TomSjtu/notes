# ffmpeg

- 容器(container)格式：一种文件封装类型，里面主要包含了流，一般会使用一个特定的后缀，比如.mov、.avi、.wav等
- 流(stream)：在容器中存储音频或者视频、字幕等数据
- 元数据(metadata)：一般位于容器之中，告诉我们一些额外的信息
- 编解码器(codec)：大部分情况下指的是一种压缩标准，比如 H.264、H.265

ffmpeg框架可以简单分为两层，上层是以 ffmpeg、ffplay、ffprobe 为代表的命令行工具，其底层是一些基础库，包含 AVFormat、AVCodec、AVFilter、AVDevices、AVUtil 等模块库。

- AVFormat：封装/解封装模块
- AVCodec：编解码模块
- AVFilter：滤镜模块
- AVDevice：设备模块
- swscale：图像转换模块
- swresample：音频转换模块
- ffmpeg：编解码工具
- ffprobe：多媒体分析器
- ffplay：播放器

## 命令行

### ffmpeg

通过`ffmpeg --help`命令可以查看 ffmpeg 常见的命令，大概可以分为以下六个部分：

- ffmpeg信息查询
- 公共操作参数
- 文件操作参数
- 视频操作参数
- 音频操作参数
- 字幕操作参数

查看支持的容器文件格式：`ffmpeg -formats`

查看支持的编解码器：`ffmpeg -codecs`

查看支持的滤镜：`ffmpeg -filters`

提取AAC音频流：`ffmpeg -i input.mp4 -vn -acodec copy output.aac`

提取H.264视频流：`ffmpeg -i input.mp4 -vcodec copy -an output.h264`

如果仅仅是使用 ffmpeg 转换封装格式，那么所使用的 CPU 资源并不多，但是编码转换会消耗大量 CPU 资源。

### ffprobe

ffprobe 支持的参数比较多，下面是一些常用的选项：

- `-v`：指定输出的详细程度，0为最少，9为最多
- `-show_format`：查看文件的容器信息，包括格式、时长、码率等
- `-show_streams`：查看文件的流信息，包括类型、帧率、分辨率等
- `show_chapters`：查看文件的章节信息
- `show_packets`：查看文件的数据包信息
- `show_frames`：查看文件中的视频帧信息
- `of`：指定输出的格式以便后续处理，比如`-of json`输出json格式

### ffplay

ffplay 不仅是播放器，同时也是测试 ffmpeg 的 Codec 组件、Format 组件以及 Filter 组件的可视化工具，并且可以做可视化的媒体参数分析。

## 封装与解封装

ffmpeg 支持多种容器格式，所谓容器就是一种允许将单个或多个音频、视频、字幕等数据流存入单个文件的文件格式，通常伴随着用于识别这些数据流的元数据。元数据被存储在头部，也被称为音视频关键数据索引。

容器格式的设计至少要满足以下几个需求：

- 捕获视频图像、音频信号
- 文件的交换与下载，包括增量下载与播放
- 本地播放
- 编辑、组合、快速定位与搜索
- 流式播放及拉流录制

### MP4

MP4 是一种常见的视频容器格式，跨平台性比较好。它主要包含两个部分：

- 元数据(moov)：包含通用信息，比如每个音频、视频帧的时间信息和偏移量等
- 音视频数据(mdat)：包含视频和音频帧，通常以交错顺序存放

MP4 文件格式中包含多个子容器，被称为 box  或者 atom，moov 和 mdat 都是 box。

关于 MP4 格式信息，需要知道以下几个概念：

- MP4 文件由多个 box 和 FullBox 组成
- 每个 box 由 header 和 data 两部分组成
- header 包含了整个 box 的 size 和 type，当 size 等于0时，代表这是最后一个 box；当 size 等于1时，代表需要一个 largesize 来描述；当 type 为 uuid时，代表这个 box 中的数据时用户自定义扩展类型
- FullBox 是 box 的扩展，在 header 中增加8位的 version 标志和24a位的 flags 标志
- data 是 box 的实际数据，可以是纯媒体数据，也可以嵌套 box，这时该 box 又被称为 container box

### 视频文件切片

视频文件切片是指将一个大的视频文件分割成多个小段的视频文件的过程。这个过程在视频编辑、分发和流媒体传输中非常有用。

下面是一个命令示例：

```
ffmpeg -i input.mp4 -c copy -map 0 -segment_time 00:10:00 -f segment output%03d.mp4
```

该命令将输入视频切片为 10 分钟的片段，并且自动输出文件名。

在 ffmpeg 中，经常使用 -ss 和 -t 参数来指定切片的起始时间和持续时间。

- `-ss`：指定切片的起始时间，单位为秒
- `-t`：指定切片的持续时间，单位为秒

示例：

```
ffmpeg -i input.mp4 -c copy -ss 00:05:00 -t 00:10:00 output.mp4
```

该命令将从第 5 分钟开始截取视频，共截取 10 分钟，输出文件名为 output.mp4。

## 编码与转码

ffmpeg 本身并不支持 H.264 编码器，而是通过集成第三方模块的方式进行支持，比如 x264，可以通过`ffmpeg -h encoder=libx264`命令查看支持的编码器。

x264 支持许多参数，可以通过`ffmpeg -h encoder=libx264`查看，这里只介绍一些重要的参数。

1. 编码器预设参数 preset：用来权衡压缩效率和编码速度，默认为 medium。 preset 参数会影响后续的很多参数，一般优先设置
2. 编码优化参数 tune：直播推流时一般选择 zerolatency
3. profile 和 level：profile 用来指定编码级别，level 用来指定编码速度
4. 码率控制：通过三个参数`-b:v`、`-maxrate`、`-minrate`来控制码率

## 流媒体技术

流媒体是指以连续的方式从一个源头传递和消费音视频数据的技术，在传输过程中很少或没有中间存储。

流式传输就是人们在互联网观看视频(包括听音频)时使用的数据传输方法。而实时流式传输是指流媒体视频通过网络实时发送，无须预先录制和存储。

实时流式传输的主要过程如下：

1. 采集：采集摄像头、麦克风、屏幕等输入源的视频和音频数据
2. 编码：对视频和音频数据进行编码，编码后的视频和音频数据称为流媒体数据
3. 分段：将编码后的流媒体数据分段，并将分段数据发送至服务器
4. CDN：内容分发网络(Content Delivery Network)将内容分发至用户
5. 解码：用户的媒体播放器(专门的应用或者是浏览器内的视频播放器)将数据解码，并播放视频

ffmpeg 支持 RTMP 流、 RTSP 流、HTTP 流、TCP/UDP 流等多种流媒体协议。

