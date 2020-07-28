---
layout:     post
title:      机器视觉实时分析第一步之取流与推流
subtitle:   在TX2上基于GStreamer推流pipeline小结
date:       2020-07-28
author:     Ganglin
header-img: img/post-bg-3.jpg
catalog: true
tags:
    - CV
    - GStreamer


---


# 在TX2上基于GStreamer推流pipeline小结

最近研究了一段时间的GStreamer，略有心得，把收获记录下来以供参考。

## GStreamer简介

GStreamer 是一个创建流媒体应用程序的框架，支持Windows，Linux，Android， iOS等平台，该框架设计是基于插件的，应用程序可以通过管道（Pipeline）的方式，将多媒体处理的各个步骤串联起来，达到预期的效果。每个步骤通过元件（Element）基于GObject对象系统通过插件（plugins）的方式实现，使得编写任意类型的流媒体应用程序成为了可能 。

## Pipeline介绍

Gstreamer的基本概念如元件（elements）、衬垫（pads）、箱柜（bins）、总线（bus）和管道（pipelines）等都是GStreamer的基本概念，不一一展开。Pipeline继承自bin，是一种特殊的箱柜，pipeline会为其内部所有的element选择一个相同的时钟，同时还为应用提供了bus系统，用于消息的接收。

```
GObject
    ╰──GInitiallyUnowned
        ╰──GstObject
            ╰──GstElement
                ╰──GstBin
                    ╰──GstPipeline
```

一个典型的pipeline如下，读取本地文件，两个线程分别处理音频和视频数据，最后通过sink元件显示。

![3-1.JPG](https://i.loli.net/2020/07/28/z2xrmK5YMhBn1AN.jpg)

## 可推流的完整Pipeline

任务是需要完成读取1080P摄像头并实时推流。参考了很多前人的工作，发现有些pipeline在我的TX2上没法跑出来，另外arm架构下才可以用gst-omx硬件加速插件。我的环境是Ubuntu18.04 + GStreamer1.14，通过不断测试，以下管道是能顺利跑通的。

**1、摄像头直接推1080P流到RTMP服务器，延时10秒**

```shell
gst-launch-1.0 -v v4l2src device=/dev/video0 ! videoconvert ! omxh264enc ! 'video/x-h264, stream-format=(string)byte-stream' ! queue ! h264parse ! flvmux ! rtmpsink location='rtmp://*******'

```

**2、摄像头直接推720P流到RTMP服务器，延时5秒**

```shell
gst-launch-1.0 -v v4l2src device=/dev/video0 ! autovideoconvert ! omxh264enc ! 'video/x-h264, stream-format=(string)byte-stream ,width=1280, height=720' \
!  queue ! h264parse ! flvmux ! rtmpsink location='rtmp://192.168.10.166:1935/live/rfBd56ti2SMtYvSgD5xAV0YU99zampta7Z7S575KLkIZ9PYk'
```

**3、jpegenc方式推送1080P到本地延迟0.29秒**

```shell
UDP发送：
gst-launch-1.0 v4l2src ! video/x-raw,width=1920,height=1080 ! jpegenc ! rtpjpegpay ! udpsink host=127.0.0.1 port=5022
UDP接收：
gst-launch-1.0 udpsrc port=5022 ! application/x-rtp,encoding-name=JPEG,payload=26 ! rtpjpegdepay ! jpegdec ! xvimagesink name=sink  force-aspect-ratio=false
```

**4、x264enc方式推送1080P到本地延迟1.16秒；用一个TX2发，一个TX2接，延时0.65秒**

```shell
发送：
gst-launch-1.0 v4l2src device="/dev/video0" ! video/x-raw,width=1920,height=1080 ! videoconvert ! x264enc tune=zerolatency ! rtph264pay ! udpsink host=127.0.0.1 port=5022
接收：
gst-launch-1.0 udpsrc port=5022 ! application/x-rtp,encoding-name=H264 ! rtpjitterbuffer latency=0 ! rtph264depay ! avdec_h264 ! videoconvert ! ximagesink
```

**5、将rtp流推送转发至rtmp服务器,很卡，且延时12秒**

```shell
发送：
gst-launch-1.0 v4l2src device="/dev/video0" ! video/x-raw,width=1920,height=1080 ! videoconvert ! x264enc tune=zerolatency ! rtph264pay ! udpsink host=127.0.0.1 port=6666
接收：
gst-launch-1.0 -v udpsrc port=6666 ! application/x-rtp,payload=100,encoding-name=H264 ! rtph264depay ! video/x-h264, framerate=30/1 ! h264parse ! avdec_h264 ! videoscale ! video/x-raw ! \
queue ! videoconvert ! omxh264enc ! 'video/x-h264, stream-format=(string)byte-stream' ! h264parse ! flvmux ! rtmpsink location='rtmp://*******'
```

**6、读本地1080p.h264文件推本地卡顿,延时4秒；读本地720P的h264视频，推送本地效果好，延时不到一秒**

```shell
发送：
gst-launch-1.0 filesrc location=1080p_10fps.h264 ! h264parse ! omxh264dec ! nvvidconv ! omxh264enc ! 'video/x-h264, stream-format=(string)byte-stream' ! rtph264pay ! udpsink host=127.0.0.1 port=5033
接收：
gst-launch-1.0 udpsrc port=5033 caps='application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264' ! rtpjitterbuffer latency=0 ! rtph264depay ! avdec_h264 ! videoconvert ! ximagesink
```

**7、推送本地1080p.h264文件到rtmp服务器，偶现画面卡顿疵掉，延时14秒；720p延时约5秒，流畅**

```shell
gst-launch-1.0 filesrc location=1080p_10fps.h264 ! h264parse ! omxh264dec ! nvvidconv ! omxh264enc ! 'video/x-h264, stream-format=(string)byte-stream' ! rtph264pay ! udpsink host=127.0.0.1 port=5000
gst-launch-1.0 -v udpsrc port=5000 ! application/x-rtp,payload=100,encoding-name=H264 ! rtph264depay ! video/x-h264, framerate=30/1 ! h264parse ! avdec_h264 ! videoscale ! video/x-raw ! queue ! videoconvert ! omxh264enc ! 'video/x-h264, stream-format=(string)byte-stream' ! h264parse ! flvmux ! rtmpsink location='rtmp://192.168.10.146:1935/live/rfBd56ti2SMtYvSgD5xAV0YU99zampta7Z7S575KLkIZ9PYk'

```

**8、最终采用方案**

软解码：用一个TX2发，一个TX2接，延时0.32-0.46秒，CPU占用率超过200%

```shell
发送：
gst-launch-1.0 v4l2src device="/dev/video0" ! video/x-raw,width=1920,height=1080 ! videoconvert ! x264enc tune=zerolatency speed-preset=ultrafast ! rtph264pay ! udpsink host=127.0.0.1 port=5022
接收：
gst-launch-1.0 udpsrc port=5022 ! application/x-rtp,encoding-name=H264 ! rtpjitterbuffer latency=0 ! rtph264depay ! avdec_h264 ! videoconvert ! ximagesink
```

硬解码：CPU占用下降到8%，720P延时0.24-0.45秒，1080P延时0.59-0.94秒

```shell
发送：
gst-launch-1.0 v4l2src device="/dev/video0" ! video/x-raw,width=1280,height=720 ! videoconvert ! omxh264enc ! 'video/x-h264, stream-format=(string)byte-stream' ! rtph264pay ! udpsink host=127.0.0.1 port=5555
接收：
gst-launch-1.0 udpsrc port=5555 ! "application/x-rtp" ! rtph264depay ! h264parse ! omxh264dec ! nveglglessink
```

尝试了无数pipeline，大部分都看不到图像，能看到摄像头正确显示且低延时，感觉不错的。

![3-2.JPG](https://i.loli.net/2020/07/28/VjcewagvkzxBbDL.jpg)

